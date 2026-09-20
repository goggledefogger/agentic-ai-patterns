---
type: pattern
date: "2026-07-20"
source: The course — one workstream store rendered as a comprehensive STATUS.md (AI) and a one-line STATUS-brief.md (human), plus native Board vs Table project views
tags:
  - architecture
  - views
  - human-readable
  - github-projects
  - presentation
---

# Dual-Audience Views Over One Store (Human Brief vs AI-Comprehensive)

An AI agent wants **all** the context — every report, note, timestamp, and finding. A human wants the opposite: a tight, low-noise read. The mistake is to fork the *store* into a "human" copy and an "AI" copy. Two stores drift. Instead keep **one comprehensive store** and derive **audience-shaped views** on top of it.

## The Pattern

- **One store, comprehensive.** The store (issues + status comments + fields) holds everything the AI needs. Do not trim it for human comfort — the trimming happens in the view, not the store.
- **Derive a view per audience from the same parser.**
  - *Human:* a filtered, minimal presentation — a native Board view showing a few fields, plus a generated one-line-per-item brief (`STATUS-brief.md`) sitting next to the comprehensive dump (`STATUS.md`).
  - *AI / comprehensive:* the raw store plus the full generated view.
  Build the model once, render it twice, and add a test asserting the two renders can't disagree (the brief resolves state the same way the comprehensive one does).
- **Leverage native platform separation — it's often free.** GitHub Project boards render **fields only, never comment bodies**. So the noisy AI context (which lives in comments) is structurally off the human board with zero effort. Learn your platform's built-in views/filters/field-visibility/slice *before* building anything; the audience split is frequently a saved view away, not a project.
- **Don't bake human formatting into the store.** Presentation lives in views and (later) a front-end; the store stays canonical and structured so *any* future renderer — a nicer human UI included — can read it.

## Why it works

No drift (one store, many renders). Humans get signal; the AI gets everything; neither is starved. And because the store stays structured, a future tailored front-end is additive rather than a migration (`external-store-as-source-of-truth.md`).

## Watch-outs

- **A filtered human view needs its filter actually set.** A board with no filter shows every item (all your old cards mixed in), which reads as clutter and defeats the point. Set the label/status filter, or link the specific items instead of the whole board.
- **View configuration may be UI-only.** In GitHub Projects, views (filter, grouping, layout, field visibility) cannot be created or set via API — provision them once by hand. Know this before promising a script that "sets up the views."

## When to Use

- One dataset, two readers with opposite noise tolerance (an AI that wants everything, a human who wants a glance).
- You're tempted to maintain a separate "human-friendly" copy of AI-facing state.

## When NOT to Use

- A single audience — one well-shaped view is enough, don't build the second.

## Adjacent Patterns

- `memory-substrate-selection.md` — structured source of truth + regenerable compiled view; this extends it to *multiple* audiences over the one store
- `external-store-as-source-of-truth.md` — the store these views render
- `vault-as-cms-publisher.md` — the same one-store-many-renders discipline, vault-side
- `living-doc-refresh-ritual.md` — keep the generated views from drifting by always rebuilding from the store

## Source

- The course `status`: `STATUS.md` (every reporter/note/date, for the AI and auditing) and `STATUS-brief.md` (one line per workstream: resolved state, owner, latest note, link) generated from the same parser, plus a native Board view (minimal fields, for the co-instructor) plus a Table view (all fields). GitHub boards never show comment bodies, so the deep context is separated for free.
