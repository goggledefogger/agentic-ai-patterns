---
type: pattern
date: "2026-08-09"
source: The personal agent session 2026-08-09 — asked for phone links to two generated reports, the agent wrote a markdown-rendering web server while the target vault already shipped a per-item HTML renderer, seven rendered exports, and a dashboard already listed in the agent's own front-end registry
tags:
  - agent-safety
  - prior-art
  - capability-contracts
  - tool-design
  - duplication
---

## The incident

An agent was asked to make two freshly generated reports viewable on a phone. It wrote a small web server: markdown rendered on the fly, a path-traversal guard, theme-aware CSS, a registry row, a commit. The work was competent and it shipped.

It was also redundant. The resource being served already had its own renderer, seven items already carried generated HTML dashboards in an exports directory, and a top-level dashboard was **already a row in the agent's own front-end inventory** — a file the same agent had edited earlier in the same session, to add a row for the viewer that should never have existed. The human caught it in one line: *"didn't we have a better UI for this? don't invent something, just see what we had before."*

## The Pattern

Before the first line of anything a human will look at, check three places, in this order:

1. **The surface inventory** — whatever registry, index, or docs page lists existing entry points. If the project keeps one, it is the cheapest possible check and it is authoritative.
2. **The resource's own `scripts/`** — a `render-*`, `build-*`, `publish-*`, or `export-*` script means the surface exists and has a defined generator.
3. **The resource's output directory** — `_exports/`, `dist/`, `public/`, `site/`. Existing artifacts are proof of an existing surface, even when nothing documents it.

And the half that actually prevents recurrence:

> **When the surface exists but you cannot run its generator, that is a capability gap to name and route — never a licence to improvise a substitute.**

## Why it stays invisible

Two mechanisms, and the second is the one that does the damage.

**The inventory rule fires at the wrong time.** Most projects that keep a front-end registry state the obligation as *"any session that ships a surface adds a row."* That is a duty on the session that builds, so it runs *after* the build, when the sunk cost is already paid and the natural next move is to document the new thing rather than discover the old one. A check that only executes downstream of the mistake cannot prevent the mistake.

**The precipitating cause is a capability gap, not ignorance.** The agent above did not forget the renderer existed; it could not *run* it, because the generator needed a shell inside a sandboxed root where shell access was deliberately denied. Blocked from producing the real artifact, it produced the only artifact it could. And because that substitute was itself a real piece of engineering, it looked like progress rather than like a workaround — which is precisely what kept anyone from noticing it was the wrong thing to be making. Blocked, improvise, and the improvisation hides the original need.

## Watch-outs

- **The redundant thing may turn out useful, and that does not retroactively justify it.** In the recorded incident the viewer earned its keep — it reads any report folder, and it renders notes before their exports exist. A wasteful build that becomes load-bearing is still a wasteful build; treating the salvage as vindication is how the reflex fails to form.
- Editing the inventory to add your new row, without having read it first, is the specific tell. If you are touching that file at the *end* of building a surface, you should have opened it at the start.
- "There's no dashboard for this" is a claim about your search, not about the repo. It needs the same evidence as any other claim.

## When to Use

- Any request phrased as "let me see X", "make X viewable", "send me a link to X"
- Before writing a server, a static-site step, a report renderer, or a new HTML entry point
- When a generated artifact is expected but missing — check whether the generator exists before concluding the surface does not

## When NOT to Use

- The existing surface genuinely does not answer the question being asked (different audience, different data, different device) — but say which existing surface you checked and why it does not fit, rather than skipping the check
- Throwaway one-shot inspection that will not be committed or linked

## Adjacent Patterns

- `scan-prior-art-before-building-infra` — the sibling, and the same reflex one domain over. That pattern scopes itself to infrastructure capability (signing, auth, secrets, deploy, caching, retries, queuing); this one covers rendered surfaces, which its "When to Use" list does not reach.
- `a-capability-contract-must-name-the-verb` — the upstream cause. An agent with no named route to the finishing step "produces the only artifact it can", which is exactly the improvisation described above.
- `lawful-write-surface` — the correct resolution once the gap is named: a deterministic writer with a fixed command and a validated argument, rather than widening the agent's tools so it can choose commands.
- `vault-as-cms-publisher` — a third resolution worth remembering: wire generation to a commit hook and take the agent out of the generator step entirely.

## Source

The personal agent session 2026-08-09. Checked against the library before writing: greps for dashboard, frontend, rebuild, duplicate, surface, reimplement, recreate and "already exists" phrasing returned no pattern on this failure mode.
