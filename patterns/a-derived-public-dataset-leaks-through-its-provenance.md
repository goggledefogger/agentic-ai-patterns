---
type: pattern
date: "2026-10-07"
source: A 159-item public catalogue built by an agent from private notes, resumes and planning documents was swept before publishing, and the leaks sat in the metadata rather than the visible text
tags:
  - privacy
  - publishing
  - data
  - checklist
---

# A Derived Public Dataset Leaks Through Its Provenance

When an agent builds a public dataset out of private sources, the visible text is the part everyone checks. The leaks live elsewhere: in the fields that say where each fact came from, in phrasing that quotes a source, in items that were never meant to be exhibits, and in the files the build left lying around.

## The Problem

An owner had an agent build a public catalogue of 159 projects. The sources were private: a personal notes vault, 28 versions of a resume, a portfolio document, internal planning notes. The descriptions read fine. A sweep before publishing found the problems somewhere else.

- Every item had a `sources` array, and many named resume files. Read together, those filenames listed 9 companies the owner had applied to
- Facts said "per the 2026 resume", which tells a reader the resume exists and roughly what is in it
- Vault names appeared as sources, which maps the owner's private setup
- Internal planning had become exhibits: roadmaps, team weeks, contract staffing numbers
- An entry for a co-founded product carried architecture and ownership details the co-owner had never agreed to publish
- Private individuals' full names sat in band member lists and client stories
- 14 links pointed at repos that were private or gone
- Intermediate build files and already-processed drop files still held the raw sources
- Screenshots committed in the repo showed exhibit text, so fixing the data did not fix the images
- Git history held all of it

None of this was in the field anyone was reading.

## The Pattern

Sweep a derived dataset against a list, not by reading it.

1. **Provenance fields.** `sources`, `ref`, `from`, `notes`. Strip them or reduce them to a neutral kind ("a resume", "a notes vault")
2. **Phrasing that cites a source.** Search for "per the", "according to", "from my". Rewrite as the plain fact
3. **Internal planning.** Roadmaps, staffing, budgets and team rituals are work in progress, not exhibits
4. **Co-owned material.** Anything another person co-owns stays out until that person agrees
5. **Individuals' names.** People who did not choose to be public get a role, not a name
6. **Link visibility, checked by a tool.** For each repo link, ask the host (`gh repo view OWNER/NAME --json visibility`). A link that looks public is not evidence
7. **Build artifacts.** Intermediate files, caches, drop folders. They often hold the sources verbatim
8. **Images with text.** Screenshots and generated cards repeat whatever the data said when they were taken
9. **History.** If anything above was ever committed, the fix is a new root commit, not a delete

Removal is a move, not a deletion. Items that cannot go public go to a private folder beside the repo, so the owner's own copy stays complete.

## Why It Works

An agent that builds from sources does the right thing for a private tool. It keeps provenance so facts can be checked. That same diligence is the leak once the output goes public. Metadata, artifacts and history are written by the build, not by the author, so the author never reads them. A fixed list is how you look at parts of the dataset you did not write yourself.

## Watch-outs

- A grep for names finds names you thought of. Filenames and phrasing leak things nobody would think to search for, like a list of employers
- Fix the images after the data, or they will show the old text
- Checking links once is not enough. A repo can go private or vanish after publishing. Re-run the check before each release

## Adjacent Patterns

- [[sensitivity-tiered-access-control]] is the tier model the sources carried before they were mixed into one public file
- [[router-worker-exfil-containment]] is the general case: a worker that reads private material should return a summary, and a provenance field is a summary that says too much
- [[ship-the-public-catalogue-with-a-private-overlay-beside-it]] is where removed items go
- [[going-public-starts-from-an-orphan-commit]] handles the history

## Source

A public catalogue of 159 projects, swept on 2026-10-07 before and after going public. The sweep moved planning items, co-owned detail and private links into a private sibling folder and stripped the provenance fields.
