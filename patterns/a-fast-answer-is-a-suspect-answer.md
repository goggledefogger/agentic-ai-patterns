---
type: pattern
date: "2026-08-01"
source: The agent's own repo, local vision captioning bake-off on a 16GB MacBook Pro — qwen2.5vl:3b under Ollama, same image at two resolutions. Measured, not theorised.
status: validated
tags:
  - local-models
  - testing
  - anti-pattern
  - verification
  - silent-failure
---

# A Fast Answer Is A Suspect Answer

Some systems degrade by **truncating their input** rather than raising an error. When they do, the failure path does *less* work than the success path — so it comes back **faster**, and it comes back with output that looks fine.

Measured, same model, same prompt, same picture:

| Input | Time | Output |
|---|---|---|
| 768px, 126 KB | **6.9s** | *"A stunning view of the Carina Nebula, showcasing its vibrant orange and red clouds against a backdrop of countless stars."* + correct category |
| 3600px, 5.1 MB | **0.8s** | *"A stunning view"* |

The oversized image overflowed the context window. Ollama truncates silently on overflow, so nothing errored. What came back was a grammatical English fragment — the kind of string a pipeline stores as a caption and never looks at again.

**Eight times faster, and wrong.** That inversion is the whole pattern.

## Why the usual guards miss it

Every cheap assertion passes on this output:

- `if not result:` — it's non-empty
- `assert isinstance(result, str)` — it is
- a schema check for "a short string" — it is one
- a human skimming the log — *"A stunning view"* reads like a caption

The output is not malformed. It is a **truthful description of the first fraction of the input**, which is exactly what makes it indistinguishable from a real answer at the type level. You cannot catch this by inspecting the value alone.

## The two signals that do catch it

**1. Latency below the success band.** Once you know the healthy path takes ~7s, a 0.8s completion is not a lucky cache hit — it is the system telling you it skipped the work. Record the expected band and assert against it. This is the cheaper of the two checks and the more general one: it needs no knowledge of what a correct answer looks like.

**2. Structural completeness of the output, not its presence.** Ask the producer for something *shaped* — a required trailing field, a category line, a closing token — and assert the shape arrived. Here the healthy answer ended with `CATEGORY: Scene` and the truncated one had no category at all. A guard testing for the required field catches it; a guard testing for non-empty text never will.

Use both. Latency catches the ones whose shape you can't specify; shape catches the ones that are slow *and* wrong.

## Where else this shows up

Anywhere a component silently drops input it cannot hold, and the drop makes it cheaper:

- context-window overflow in any local model runtime (see [`local-model-agentic-tool-calling.md`](local-model-agentic-tool-calling.md) — Ollama truncating to 4K is the same mechanism, surfacing as "tool-calling broke")
- a query that hits a `LIMIT` and returns the first page as if it were the whole set
- a paginated API whose second page fetch fails and returns page one
- a cache that returns a stale partial on a miss instead of fetching

The unifying tell is not the output. It is that **the system got cheaper at the moment it got wrong**, and cost is a signal you are usually not watching.

## The rule

> When a component can fail by doing less work, its latency is part of its contract. Assert on the band, and on the shape of what came back — never on whether *something* came back.

Related: [`unrun-checks-read-as-passing.md`](unrun-checks-read-as-passing.md) (a check that never ran reads as passing — this is its sibling, where the check *did* run and passed on garbage), [`health-check-that-never-exercises.md`](health-check-that-never-exercises.md), [`verification-needs-a-negative-control.md`](verification-needs-a-negative-control.md).
