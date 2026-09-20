---
type: pattern
date: "2026-08-12"
source: A work vault, 2026-08-12; a person wikilink written from a source that gave only a first name, the surname invented to satisfy the slug format, caught by a post-write verification grep. A sweep then found 28 more of the same shape already in the vault
tags:
  - obsidian
  - wikilinks
  - agents
  - hallucination
  - schema
  - verification
  - hooks
---

# A Required Slug Invents the Missing Half

A convention that says "always link people" plus a link format that needs `firstname-lastname` equals a standing demand for a surname. When the source says only "Jordan," the format still needs the other half, and a plausible one is free to produce. The output names a person who does not exist and *reads as verified*, because a wikilink looks like a resolved reference. Fix is a cheap mandatory lookup before writing any person link, plus a near-miss detector that fires when a real first name carries a surname that resolves to nothing.

## The Problem

Three ordinary, individually-correct decisions compose into a name generator:

1. **The vault convention:** link every person, every time, `[[slug|Display Name]]`.
2. **The slug format:** `firstname-lastname`, because two people share a first name eventually.
3. **The source:** conversational. Slack messages, transcripts, meeting notes and spoken drafts say "Priya" and "Jordan," not full names.

Writing the link requires a fact the source never supplied. An agent mid-draft does not experience this as a missing input — it experiences a slot, and slots get filled. `jordan-quinn` is a perfectly ordinary surname attached to the right first name, and nothing in the rendered output distinguishes it from a link that resolves.

**The failure is invisible at exactly the moment it matters.** A wikilink is the vault's marker for *this reference is real and resolvable*. Wearing that marker, a fabricated person inherits the credibility of every correct link on the page. In Obsidian it renders as an unresolved link — a visual signal that exists but that nobody is looking at while reading prose, and that never reaches anyone consuming the file as text (an agent, a grep, an export, a pasted excerpt).

### The verification asymmetry that makes it survive review

The same draft got one name right and one wrong, and the mechanism explains why:

- `priya-nair` — **verified.** His file had been read minutes earlier for an unrelated check, so the slug was a real filename that had passed through the session.
- `jordan-quinn` — **invented.** He had appeared only as a first name inside filename fragments like `coaching-jordan-1on1-2026-08-11`, and in prose. No surname had ever been observed.

In working context these two feel identical. Both names are familiar, both belong to real colleagues, both were about to be linked with equal confidence. **"I have seen this name" and "I have seen this slug" are different claims, and nothing in the reading experience separates them.** That is the actual defect — not carelessness, but an epistemic distinction that leaves no trace.

### It is not the transcription-error family

Vaults that process meeting transcripts usually already have rules for bad names: cross-check against a glossary, verify pronouns and thread continuity, disambiguate two people sharing a first name. Every one of those handles a name that arrives **garbled or ambiguous**.

This name arrives **correct but incomplete**, and gets completed by the writing convention. No existing check fires, because there is nothing suspicious about the input — "Jordan" is exactly right. Anti-hallucination instincts calibrated on *wrong* input do not trigger on *insufficient* input.

### Evidence that it is structural

A one-off would not justify a hook. The sweep found **28 distinct instances**, overwhelmingly in durable files rather than scratch:

| Written | Real person | Count |
|---|---|---|
| `dana-gill` | `dana-ellis` | 6 |
| `indraneel-makam`, `indraneel-mane` | `indraneel-purohit` | 3 |
| `sharon-stoffel`, `sharon-hopkins` | `sharon-lu` | 2 |
| `mika-burton`, `mika-adeyemi` | `mika-carino` | 2 |
| `casey-brandt` | `casey-moreno` (surname borrowed from `andre-brandt`) | 1 |
| `simi-shenoi` | `simi-damani` | 1 |

`casey-brandt` is the instructive one. It is not a random surname, it is a real surname belonging to a *different colleague in the same vault* — the generator reaches for locally plausible material, which is precisely what makes the result survive a proofread.

## The Pattern

### 1. A slug is real only if you saw it as a filename

Before writing any person wikilink whose slug you have not observed as an actual file this session:

```bash
ls company-context/team/ company-context/contacts/ recruiting/candidates/ | grep -i <firstname>
```

One command, deterministic answer, no judgment. The rule is deliberately mechanical because the failure is not a reasoning failure — reasoning is what produces the plausible surname.

State it as an epistemic rule, not a diligence rule: **seeing a first name in prose or inside a filename fragment proves nothing about the surname.** Only a directory listing, a file read, or a glob hit counts.

### 2. Put the rule where the convention that causes it lives

The guard belongs immediately after the "always use wikilinks for people" rule in the always-loaded context file, not in a separate style guide. The convention creates the pressure; the counterweight has to be read in the same breath. A rule filed somewhere else is a rule that arrives after the link is written.

### 3. Detect near-misses, not broken links

The mechanical signal is sharp: **the first segment matches a real person file, the full slug matches nothing.**

```python
stems = {p.stem for p in root.rglob("*.md") if ".git" not in p.parts}
by_first = {}
for d in PERSON_DIRS:
    for p in d.glob("*.md"):
        by_first.setdefault(p.stem.split("-")[0], []).append(p.stem)

for slug in wikilinks_in(content):
    if "-" in slug and slug not in stems and slug.split("-")[0] in by_first:
        warn(slug, did_you_mean=by_first[slug.split("-")[0]])
```

This is precise in both directions. It fires on `jordan-quinn` and stays silent on `[[gpt-oss]]` or `[[acme-co]]`, which are two-segment lowercase slugs that merely look person-shaped, because no person file starts with those segments.

**Do not warn on all broken links.** The same vault carried ~180 bare first names (`[[mika]]`) and stale suffixes (`[[taylor-reed-candidate]]`). Both are genuinely broken and neither invents a human being, and a check that reports all of them buries the one that matters under pre-existing debt. Scope the alarm to the failure with a victim.

### 4. Run it as a PreToolUse hook, warning not blocking

The check costs ~35ms over 2,300 files, cheap enough per write. Warn rather than block: a legitimate forward reference to a not-yet-created person file exists, and blocking would make the hook something to route around.

If a person-adjacent hook already exists, extend it rather than adding a sibling. In the source vault a pronoun-conflict hook was already extracting wikilink slugs and resolving them against person directories — and its `load_pronouns()` already returned `None` for a slug with no file, then `continue`d past it. **The detection was already being computed and thrown away.** One collection step converted it into a check.

## Why It Works

- **The lookup is cheaper than the deliberation it replaces.** Any rule of the form "be careful about names" competes with the drafting task for attention and loses. `ls | grep` does not.
- **A near-miss carries its own correction.** The warning does not say "this might be wrong," it says *did you mean `jordan-vance`*, so resolving it costs one keystroke rather than an investigation.
- **It converts a judgment failure into a mechanical one.** Like greppable forecast verbs in `silence-is-not-confirmation`, the composite-key near-miss is one of the rare hallucination shapes with an exact detector, because the vault already contains ground truth about which people exist.

## Watch-outs

**This generalizes past wikilinks to every required composite key.** Anywhere a schema demands a field the source did not supply, the same substitution happens: citation keys (`author2024title`), module paths in generated imports, required frontmatter fields, IDs in structured-output schemas, filenames built from metadata. **A required field is a standing request for information, and a generator will satisfy it from priors rather than decline.** Wherever a format composes a key from parts, ask which part the source actually supplied.

**Optional fields do not have this problem, which suggests a design lever.** Had the convention permitted `[[jordan]]` for a not-yet-verified reference, the incomplete state would have been representable and the invented surname unnecessary. When a format has no way to say "I only know half of this," it will be told a whole thing that isn't true.

**A verification pass that runs after the write catches it, and that is not enough.** The original instance was caught by a post-hoc grep, which is real but relies on the same agent choosing to check. The hook moves it before the write, where it does not depend on the writer's judgment.

**The near-miss detector cannot see an entirely fabricated first name.** It anchors on a real first-name segment. A wholly invented person passes silently — for that, only "did you observe the file" holds.

**Historical instances need human review, not a bulk rewrite.** Several of the 28 are ambiguous (two real people share the first name), and a scripted fix would pick wrong. Report them, let a person adjudicate.

## How to Adopt

1. Run the sweep and see whether you already have the problem. If your vault has more than a couple of person files and any transcript processing, you probably do.
2. Add the rule to the always-loaded context file, directly beneath the wikilink convention, with the `ls | grep` command written out literally.
3. Add the near-miss check to an existing person-aware PreToolUse hook if one exists, or as a small standalone one. Warn, do not block.
4. Test the exact failure plus the three silences: correct slug, bare first name, non-person link. A check that fires on all four is worse than none.
5. Report existing instances to the vault owner as a list with suggested corrections. Do not bulk-fix.
6. Ship the rule, the hook, and the hook's README entry in the same commit. A hook nobody knows about gets deleted during the next cleanup.

## Adjacent Patterns

- **wire-into-existing-flows** — the rule ships in the always-loaded context file next to the convention that causes it, which is the only forcing function read every turn.
- **wikilink-graph-hygiene** — the other wikilink-integrity pattern. That one covers which node categories deserve stubs and how bare links resolve non-deterministically across symlinked vaults. This one covers a slug that never had a referent to resolve to.
- **silence-is-not-confirmation** — sibling failure shape. There the newest note reads as current state, here a wikilink reads as a verified reference. Both derive confidence from a format's implicit promise rather than from evidence, and both turn out to have a mechanical detector.
- **implicit-identity-silent-wrong-answer** — same silence: a plausible value substitutes for a missing one and nothing in the output marks the substitution.
- **a-heuristic-where-an-exact-key-exists** — the general form. Ground truth about which people exist was on disk the whole time; a plausible-surname heuristic ran in front of an exact lookup.
- **framework-gotcha-comments** — the "why does this hook check surnames" comment belongs above the check, since the reason is not recoverable from the code.

## Source

- The work vault, 2026-08-12. A Slack draft named "Priya (or Jordan)"; converting both to wikilinks produced `[[priya-nair]]` (correct, his file had been read earlier that session) and `[[jordan-quinn]]` (invented, only the first name had ever been observed). The real file used a different surname entirely.
- Caught by a post-write verification grep, before the user saw it. The user's follow-up question — "how to avoid you making up names?" — prompted the sweep.
- Sweep: 206 distinct broken wikilink slugs vault-wide, 43 person-shaped, **28 near-misses where a real first name carried a non-resolving surname**. Concentrated in `company-context/team/` and `sources/transcripts/`, i.e. the durable, trusted layer.
- Hook: extended `.claude/hooks/check-pronouns.py` (PreToolUse on Write/Edit/MultiEdit) rather than adding a sibling, since it already extracted wikilink slugs and resolved them against the person directories, discarding the unresolved ones. Kept the filename because the hook is registered in a gitignored per-machine `settings.local.json` by absolute path, so a rename would silently disable it on another machine.
