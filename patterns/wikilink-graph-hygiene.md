---
type: pattern
date: "2026-05-03"
source: A course content-engine vault, four decision records across 2026-05-02 and 2026-05-03 on concept notes, citation-focused person stubs, and persona names left deliberately unresolvable; plus a symlink-collision finding from testing the next evening
tags:
  - obsidian
  - wikilinks
  - graph
  - schema
  - linked-vaults
  - personas
---

# Wikilink Graph Hygiene

Three rules for Obsidian vaults that have multiple categories of graph node (concepts, people, agent personas) AND that symlink another vault. (1) Decide which node categories you have and what shape each takes — the categories drive different stub policies. (2) Personas — the intentional non-resolvers — must explicitly NOT trigger your "stub on first encounter" rule. (3) When you symlink another vault, bare wikilinks resolve non-deterministically across same-named files; path-prefix any wikilink whose resolution target is load-bearing.

## The Problem

A working vault accumulates wikilinks that look identical in markup but mean very different things:

- `[[PRD]]` — a *concept*. Resolves to a graph-node note that defines the term and rolls up where it's discussed.
- `[[lee-navarro|Lee]]` — a *person citation*. Resolves to a stub used for schema author-attribution, hover-preview text, byline rendering.
- `[[mary]]` — a *persona label*. Refers to a BMad agent character whose canonical reference is a TOML config, not a markdown file.

Without explicit conventions per category, three failure modes show up in the same week:

1. **Agents auto-create wrong-shape notes.** Paige's persona principle says "create a stub when you wikilink a missing person." She wikilinks `[[mary]]`, follows the rule, creates `People/mary.md`. Now the persona has a duplicate that drifts from the TOML. (Real instance.)
2. **Symlinked-vault collisions silently degrade resolution.** When this vault symlinks another vault's `People/`, both have `People/lee-navarro.md` — different content, different purpose. Obsidian indexes both. Bare `[[lee-navarro]]` resolves *non-deterministically*. The citation-focused stub is bypassed; schema markup uses the wrong target. (Real instance, surfaced via UAT 2026-05-03 evening on `[[maya-oduya]]`.)
3. **Click-creation lands stray files.** Even when the display correctly greys out an unresolved persona link, *clicking* it creates the file at the resolved path. The community CSS workaround (`pointer-events: none` on `.is-unresolved`) was tested in this vault and did NOT block creation. Discipline is the only enforcement that holds today (May 2026).

## The Pattern

### 1. Name the categories explicitly

Decide what graph-node *categories* your vault has, give each its own folder, and write per-category stub policy into CLAUDE.md:

| Category | Folder | What it is | Schema target? | Stub-on-first-encounter? |
|---|---|---|---|---|
| Concept | `Concepts/` | Idea node. Has aliases, "What it is," "Where it appears" backlinks roll-up. | No — internal. | Yes |
| Person | `People/` | Real-human citation stub. Has aliases, "How to cite," schema author URL. | Yes — author attribution. | Yes |
| Persona | (none — TOML only) | Agent character. Canonical ref is the persona config (e.g. `_bmad/custom/<persona>.toml`), not markdown. | No — never published. | **No — exempt** |

The persona row's "no stub" cell is the load-bearing one. Without it, agents who follow the People convention auto-create persona stubs on first encounter and the TOML drifts.

CLAUDE.md is the right place to encode the table; it's the file every session re-reads. Per-category stub rules go *in* the table, not as separate prose paragraphs that decay.

### 2. Path-prefix citation-grade wikilinks in linked vaults

If your vault symlinks another vault (a common pattern when one is the canonical source for shared material and the other is a derivative):

```
this-vault/
  course-vault/             → symlink to ../course-vault
  People/
    lee-navarro.md          ← citation-focused stub (this vault)
  ...
Course-vault/
  People/
    lee-navarro.md          ← course-planning note (other vault)
```

Obsidian indexes both. Bare `[[lee-navarro]]` resolves to whichever Obsidian decided first. Empirically, the linked-vault file wins. The citation stub is bypassed silently and schema markup degrades.

Fix: *path-prefix any wikilink whose resolution target is load-bearing.* For the example above:

- **Citation contexts** (article body cite, proof frontmatter `author:`, Concept/Person "Where it appears" sections): `[[People/lee-navarro]]`. Pins resolution to this vault.
- **Casual prose contexts** (Daily notes, ROADMAP, scratch, internal meeting notes): bare `[[lee-navarro]]` is fine.

The split is "load-bearing vs. casual," not "everywhere vs. nowhere." Backfilling 100+ existing bare cites is high-friction and low-payoff (the linked-vault file is about the right person, just the wrong shape — degraded UX, not broken). The load-bearing rule covers new content where citation accuracy matters and lets old prose stay.

### 3. Persona names stay intentionally non-resolving

For persona labels (`[[mary]]`, `[[paige]]`, `[[john]]` in a BMad-style vault):

- **Do not create `People/<persona>.md` or `Personas/<persona>.md`.** The TOML is the canonical reference. Creating a markdown stub forces synchronization with the TOML or accepts drift; both lose.
- **Persona principles must include the explicit carve-out:** "the stub-on-first-encounter rule does NOT apply to persona names." Otherwise the agent that writes `[[mary]]` follows its own People-stub rule and breaks the convention.
- **No People stub may carry a persona name as an alias.** A real person genuinely named "Mary" gets `aliases: ["Mary Smith", "M. Smith"]` (fuller form) plus the path-prefixed citation rule. Bare `aliases: ["Mary"]` would silently make `[[mary]]` resolve to that person and break the persona convention with no visible signal.
- **Don't click persona wikilinks.** Obsidian's default click action creates the missing file at the resolved path. Tested CSS workarounds did not block this in current versions. If a stray gets created, `rm` it promptly. The convention is intact as long as no stray persists.

The greyed-out render is the desired signal: "this is a persona, look in the TOML."

## Why It Works

- **Three folders × three stub rules** scales: you can add categories later (Source, Project, Decision) without re-litigating each one. Each gets its own row in CLAUDE.md with its own policy.
- **Path-prefixing is opt-in for what matters.** You don't pay rewrite cost on the prose where bare cites are fine; the cost lands only on author-attribution-grade contexts.
- **Persona-as-non-resolving** uses Obsidian's render behavior as a *signal* (greyed = "look in TOML") rather than fighting Obsidian's index.

## Watch-outs

**Persona-alias collision is the silent-break failure mode.** A People stub for a real person sharing a name with a persona, with a bare-form alias entry, breaks the persona convention without any warning. Worth a vault-wide grep on each persona name when accepting new People stubs.

**Click-creation is unfixable in current Obsidian** (open forum request since 2022). Discipline-only is the state of the art (May 2026). Revisit if a working snippet/plugin/Obsidian-setting appears in a future release.

**The third-party `File Ignore` plugin** would prevent the linked-vault indexing entirely by renaming the symlinked folder to a dotfile. It also breaks every existing wikilink that depends on the linked vault being indexed (e.g., `[[course-vault/Meetings/...]]`). Usually not worth the trade.

**Excluding the linked vault from Obsidian's index** has the same break-pattern as `File Ignore`. Don't reach for it unless the project doesn't use explicit-path wikilinks into the linked vault.

**Backfill is usually skippable.** Existing bare cites in old articles are degraded but not broken. Pay the rewrite cost when public publication forces it (schema author attribution gets shipped to a public URL), not before.

## How to Adopt

1. Decide your categories. List them in CLAUDE.md as a folder-structure table with per-category stub policy in the same row.
2. If your project uses agent personas (BMad or otherwise), add the explicit carve-out: "persona names do NOT trigger stub-on-first-encounter, and No People stub may carry a persona name as an alias."
3. If your vault symlinks another vault, run a one-time grep for collisions: `find . -name "<slug>.md"` for each cited entity, or `find . -name "*.md" -type f | sed 's|.*/||' | sort | uniq -d`. Where two files share a name, path-prefix the citation-grade wikilinks in articles + proofs + extraction sheets. Skip casual-prose backfill.
4. Ship the table + the carve-out + the path-prefix rule in the same commit as the convention. Splitting them means the carve-outs get forgotten.
5. Test with the next real cite. If a person citation lands wrong, the path-prefix rule isn't being followed; if a persona stub appears in `People/`, the carve-out isn't being followed.

## Adjacent Patterns

- **wire-into-existing-flows** — these rules ship in CLAUDE.md (and matching agent-persona principles), the only forcing function read every turn.
- **decision-doc-adr** — when a category convention changes (a new collision found, a new category added), record it as an ADR appendix rather than rewriting the original. The collision-finding here was a same-day appendix to the People-note ADR.
- **framework-gotcha-comments** — a 2-line "why bare cites fail in linked vaults" comment in CLAUDE.md, near the wikilink rule, prevents the next author from writing a degraded cite.

## Source

- Conceptual model: A course content engine vault, `Concepts/` + `People/` + Personas-as-TOML scheme. Four ADRs at `docs/decisions/2026-05-02` (concepts + NotebookLM querying) and `docs/decisions/2026-05-03` (People convention, persona character homing, opener-mirror first-mention rule).
- Linked-vault collision finding: same vault, UAT 2026-05-03 evening — bare `[[maya-oduya]]` resolved to the linked course-vault course-planning note instead of this vault's citation-focused stub. Path-prefixed form `[[People/maya-oduya]]` resolved correctly. 9 of 10 People stubs in this vault had collisions in the linked vault; only one slug had no collision.
- Persona click-creation finding: same UAT — clicking a greyed `[[mary]]` wikilink from a scratch note created stray `Inbox/mary.md` in direct violation of the persona convention. Community CSS snippet using `pointer-events: none` on `.is-unresolved` selectors was tested and did not block click-creation in current Obsidian versions; the snippet was reverted in commit `e71fb24`.
