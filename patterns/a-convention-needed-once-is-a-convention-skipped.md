---
type: pattern
date: "2026-08-12"
source: A work vault; a hired candidate's file and a new team file for the same person shared a filename stem for seven weeks, with 142 bare wikilinks resolving arbitrarily between them. The suffix convention that prevents it was already documented
tags:
  - obsidian
  - wikilinks
  - conventions
  - lifecycle
  - checklists
  - audit
---

# A Convention Needed Once Is a Convention Skipped

When a record graduates — candidate to employee, lead to client, draft to published — the system often keeps *both* files, because both remain true. The graduation is what creates a name collision, and the convention that prevents it is needed exactly once, at the transition, in the middle of the one moment when everyone's attention is on the new thing. Documenting it as a filing convention guarantees it gets skipped. It belongs in the checklist that already runs at the transition.

## The Problem

A candidate has a file under `recruiting/candidates/`. They get hired, so a team file appears under `company-context/team/`. Both records stay valid and neither should be deleted: one is the recruiting history (interviews, scorecards, sprint, offer), the other is the employee record (role, pod, 1:1s, reviews).

Now two files share a stem, and in Obsidian a bare `[[sam-ortiz]]` resolves between them **non-deterministically**. Nothing errors. Nothing renders red. Links simply start landing on whichever file the index picked, and a reader following one gets a plausible, wrong document.

Three properties make this survive:

**The break is caused by an addition somewhere else.** Nobody edited the candidate file. Creating the *team* file is what broke it. No diff, no review, and no test touches the record that changed meaning — which defeats the instinct that says "check what you just modified."

**The one link that would have screamed was itself the broken one.** The new team file said `Recruiting history in [[sam-ortiz-candidate|his candidate file]]` — correct under the convention, pointing at a file nobody had renamed. So the single artifact that encoded the right answer was the single artifact that silently resolved to nothing.

**The convention was known and mostly followed.** Three prior hires — `davis-bennett-candidate`, `drishtie-patel-candidate`, `fanny-casal-candidate` — were all suffixed correctly. This is a **3-of-4 compliance rate, not 0 of 4**, and that distinction picks the fix. A rule nobody follows needs rewriting or deleting. A rule followed 75% of the time is understood and correct, and is being dropped at a predictable moment. Rewriting it harder does nothing.

### Why the once-only step is the droppable one

The suffix is *never* needed while someone is a candidate. Most candidates are never hired, so most candidate files correctly never carry it. The step becomes necessary at exactly one instant, and that instant is the busiest one in the whole lifecycle — offer signed, start date, onboarding buddy, pod assignment, access provisioning, announcement.

That combination is what makes it the most droppable kind of step:

- needed **once**, so no repetition builds the habit
- needed at a moment whose **attention is entirely on the successor record**, not the predecessor
- filed in a **conventions document**, where it reads as a timeless filing rule rather than an action with a trigger

The third is the actual defect. **A conventions doc describes what is true of the vault; a checklist describes what to do at an event.** A step with a trigger, placed among rules without triggers, has no moment that summons it. It gets read during onboarding to the vault and never again at the moment it fires.

## The Pattern

### 1. Move the step to the event checklist, keep a pointer in the conventions doc

Find the checklist that already runs at the transition — a hire-closeout list, an offboarding list, a publish routine — and put the rename in it, next to the other artifact updates. Leave a one-line pointer where the convention is stated, framed as **"the rename is a hire step, not a filing convention."**

Both locations, one commit. The conventions doc explains the shape, the checklist causes the action.

### 2. Rename and repoint in one pass, and separate history from live

```bash
git mv records/<slug>.md records/<slug>-<qualifier>.md   # git mv, so history follows
grep -rn "records/<slug>\.md"                            # then repoint path references
```

`git mv` matters: a delete-plus-add loses the provenance of the record you most want provenance on.

**Repoint live references, not historical ones.** Dated audit files, snapshots, and archived exports are records of what was true then — rewriting a path inside them makes them lie about their own moment. In the source vault this split four live references (a map, three processed transcripts) from three history files that were correctly left stale.

### 3. Write the audit as a set intersection that can fail

Collisions cannot be detected from inside either record. Both files are individually perfect. Only a check that reads the whole set sees it:

```python
team = {p.stem for p in Path("company-context/team").glob("*.md")}
print([p.stem for p in Path("recruiting/candidates").glob("*.md") if p.stem in team])
```

Put the one-liner in the checklist so it runs at the moment the risk is created, and expect `[]`. Same shape as `identity-is-a-property-of-the-set`: cross-instance uniqueness needs a cross-instance reader, and one that merely prints is a report nobody runs.

### 4. Generated artifacts regenerate, don't hand-patch them

Derived data files carrying the old path clear on their next build. Editing them by hand creates a diff that competes with the real change and desynchronizes them from their generator.

## Why It Works

- **It attaches the action to its trigger.** The step was never misunderstood, it was never *summoned*. A checklist that already runs at the transition summons it for free.
- **It costs nothing when the transition doesn't happen.** Candidates who are never hired never touch the step, which is exactly the property that made a conventions-doc home feel adequate.
- **The audit converts a silent failure into a failing check.** Non-deterministic resolution has no error surface at all, so the only way to see it is to look for it deliberately.

## Watch-outs

**This is the graduation shape, not the migration shape.** Migrating a record means moving it and updating consumers (`merge-then-update-consumers`). Graduating means the old record stays valid *alongside* a new one. Don't "fix" a collision by deleting the predecessor — the recruiting history is the reason the file exists, and it is exactly the thing you want when the same person is up for review a year later.

**A 75% compliance rate is a placement problem, not a comprehension problem.** Before rewriting a rule that keeps getting missed, check the hit rate. High-but-not-perfect compliance means the rule is understood and lives in the wrong place. That diagnosis is cheap and it points at a different fix than "say it louder."

**The suffix has to be on the predecessor, not the successor.** The successor is the record everything links to going forward, and it should own the bare name. Suffixing the new team file instead would leave every future bare link resolving into the archive.

**Bare-name resolution is convenient and it is why this bites.** The same shortest-path resolution that makes `[[name]]` pleasant to write is what makes collisions silent. `wikilink-graph-hygiene` covers the cross-vault version; this is the within-vault version, created by a lifecycle event rather than by a symlink.

**Generalizes past vaults.** Anywhere an entity graduates and both records persist: lead → client, trial → subscriber, draft → published, applicant → employee, staging record → production record. The question to ask of any lifecycle transition is *what becomes ambiguous the moment the successor exists*, and whether the step that disambiguates lives anywhere that fires.

## How to Adopt

1. Run the set intersection for every predecessor/successor folder pair you have. If it returns anything, you already have live collisions.
2. Find the checklist that runs at the transition. If none exists, that is the finding — the collision is a symptom of an unowned transition.
3. Add the rename to that checklist, beside the other artifact updates, with the audit one-liner underneath.
4. Reduce the conventions-doc entry to a pointer that names the trigger.
5. Fix existing collisions with `git mv`, repointing live path references only.
6. Record the compliance rate in the checklist note. "Applied 3 of 4 times, missed on X" tells the next reader this is a real slip, not a hypothetical.

## Adjacent Patterns

- **wire-into-existing-flows** — the general form. This is a case where the artifact existed, was correct, and was in a document that never fires at the relevant moment.
- **wikilink-graph-hygiene** — the same non-deterministic bare-link resolution, arising from a symlinked sibling vault rather than a lifecycle transition. Its path-prefix fix treats the symptom; renaming removes the collision.
- **identity-is-a-property-of-the-set** — the audit shape. Uniqueness is unverifiable from inside one instance and needs a cross-instance check that fails rather than reports.
- **a-required-slug-invents-the-missing-half** — sibling failure in the same vault, found the same day. There a link named a person who never existed, here a link named a person who exists twice. Both fail silently because a wikilink looks resolved either way.
- **merge-then-update-consumers** — the contrast case. That is a migration, where the old thing should stop existing; this is a graduation, where it must not.
- **silence-is-not-confirmation** — the shared root: a format that offers no way to represent an unresolved state renders one as settled.

## Source

- The work vault, 2026-08-12. `recruiting/candidates/sam-ortiz.md` and `company-context/team/sam-ortiz.md` had coexisted since the hire closed June 26, seven weeks, with **142 bare `[[sam-ortiz|...]]` links** resolving arbitrarily between them.
- Surfaced incidentally while auditing a different wikilink failure. The collision was invisible to the near-miss detector that found the other one, because both files exist — it took a separate set-intersection check.
- Compliance before the fix: 3 of 4 (`davis-bennett-candidate`, `drishtie-patel-candidate`, `fanny-casal-candidate` correct). The hire-closeout checklist in `recruiting/CLAUDE.md` already covered candidate-file status, the map row, the matrix, role retirement, and pool release — every artifact except the filename.
- Fix: `git mv` the candidate file, repoint four live path references, leave three dated audit/snapshot files stale on purpose, add the rename plus the audit one-liner to the hire-closeout checklist and its mirror in the post-sprint skill, and reduce the root conventions entry to a pointer naming the trigger.
