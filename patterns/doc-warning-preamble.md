---
type: pattern
date: "2026-05-01"
source: A course participant's home-agent system (the workspace repo §9 callout, 2026-04-29)
tags:
  - documentation
  - drift
  - verification
---

# Doc-with-Warning-Preamble

When a living doc has known drift potential — the refresh cadence can't keep up with the build cadence — don't try to make it fresher. Add a warning preamble at the top that names the failure mode and gives readers a verification recipe. Make the cost of staleness visible to readers instead of papering over it.

## The Problem

Architecture docs, status pages, and roadmaps drift between refreshes. The standard responses are (a) refresh more often (doesn't happen), (b) put a "last refreshed" date at the top (readers ignore it), or (c) accept the drift quietly (readers cite stale claims as fact). All three fail.

The specific failure mode that matters: an agent reads the doc, treats it as ground truth, and acts on a claim that's no longer current. The participant's instance — the explainer persona recommended building a planned subsystem three days after a planned subsystem shipped, because the brief still listed it as the top blocker. The agent didn't fail; the doc lied silently.

## The Pattern

Add a warning preamble at the top of any section where state can drift. It has three components:

### 1. Name the failure mode

Be specific. Not "this doc may be out of date." Specifically: *"Items listed in §9.3 as 'to-build' may have shipped since the last refresh."* The reader needs to know what kind of wrongness to suspect.

### 2. Give a verification recipe

Three to five concrete commands that resolve any specific claim against ground truth. Each one should be runnable without context:

- *"Cross-reference against (a) `BUILD-STATUS.md` header timestamp, (b) `git log --since=<last-refresh-date>`, (c) `ls _shared/` and `ls the workspace repo/*.md`."*

The recipe shifts cost from the writer (who can't keep the doc fresh) to the reader (who needs to verify before acting). That's the right place for it.

### 3. Reframe the doc's role

State explicitly what the doc is and isn't. The participant's framing: *"The brief is a thinking tool. The filesystem is ground truth."* That sentence does more work than any "last refreshed" date. It tells the reader the doc is a *map*, not the *territory*.

## Template

```markdown
> **⚠️ READ BEFORE TRUSTING ANY <SPECIFIC CLAIM TYPE> IN THIS SECTION.**
>
> <One sentence naming the failure mode in plain language.>
>
> Before acting on a specific claim, verify against:
> (a) <command #1 — fastest>
> (b) <command #2 — middle>
> (c) <command #3 — definitive>
>
> The doc is a thinking tool. <The runtime / the filesystem / the database> is ground truth.
> This warning was added <date> after <specific incident that motivated it>.
```

The "after <specific incident>" line is doing a lot of work. It tells future readers the warning is load-bearing — there was a real failure, not just preventive hygiene.

## When to Add

- Living-state docs (architecture, status, roadmap, what-exists indexes) that get cited as fact by agents or new team members
- Any section that lists "things that don't yet exist" — that's the highest-drift category
- Docs that have a known refresh cadence (monthly, quarterly) — drift is inevitable between refreshes

Skip for:
- Stable docs (constitutions, principle definitions, terminology) where drift isn't the failure mode
- Docs the writer can keep fresh in real-time (commit-by-commit changelogs)
- Single-author solo-use notebooks where the writer is also the only reader

## Example from the participant's system

`_bmad-output/product-brief.md` §9 (Living Layer — Current State) had a 2026-04-29 callout added after the explainer persona made the planned subsystem recommendation error. The warning names the failure mode (*"Items listed in §9.3 as 'to-build' may have shipped"*), gives the recipe (BUILD-STATUS header, `git log --since`, `ls _shared/`), reframes the doc's role (*"the brief is a thinking tool, the filesystem is ground truth"*), and dates the addition with the motivating incident.

The same brief admits its own limits in §10 Open Questions: *"the 90% coverage threshold exists as a concept but the formal definition and monitoring mechanism are not yet written. Opened: 2026-04-07. Revisit by: 2026-05-07."* Same posture: track the gap as a dated deliverable rather than papering over it.

## Adjacent Patterns

- **Living-doc refresh ritual** (`living-doc-refresh-ritual.md`) — the pattern that splits stable / living / distillate and refreshes against the running system. The warning preamble is the *next iteration*: even with a refresh ritual, drift will happen between refreshes, and the cost lands on readers. The two patterns work together — the ritual closes drift over time, the preamble protects readers in the meantime.
- **Living-doc archive-split** (`living-doc-archive-split.md`) — the bloat complement. The preamble protects readers from *stale* docs; the archive-split keeps docs *short enough to be worth reading*. A bloated doc with a warning preamble is still bloated.
- **System-understanding protocol** (`system-understanding-protocol.md`) — the reader-side complement. The doc admits its own staleness; the protocol enforces verification before acting on doc-derived claims.
- **Decision-doc ADR** (`decision-doc-adr.md`) — decision docs benefit from the same "verify before acting" line, since "what was decided" can be superseded silently by a later decision.

## How to Adopt

1. Find one the author-authored doc with known drift potential — start with the obsidian-claude guide's "State of recommendations" section, the course syllabus, or any project README listing "current architecture."
2. Write the warning preamble using the template above. Make the failure mode specific.
3. Identify three commands that verify any specific claim. Test them yourself — they should work without context.
4. State the doc's role explicitly: *"this is a thinking tool. <X> is ground truth."*
5. Date the addition. If there's a specific incident that motivated it, name the incident.
