---
type: pattern
date: "2026-07-22"
source: The household-agent repo (a household agent on a Raspberry Pi). A routing file auto-injected into every turn sat 21 characters under its host's truncation cap. A ~460-char rule pushed it over; a live loader probe showed 20,179 chars loading as 18,198 — the middle cut out, warning only to a log nobody reads. A story two months earlier had compressed that same file specifically to buy ~3,500 chars of headroom; every byte had been silently re-consumed because nothing watched the number.
tags:
  - agents
  - context
  - cost
  - reliability
  - prompt-engineering
  - claude-code
---

# Treat Always-Loaded Context Files as a Budget, Not a Place to Write

`CLAUDE.md`, `AGENTS.md`, `SOUL.md`, `.cursorrules` — files a harness injects into the system prompt on **every turn**. They feel like documentation, so they accumulate like documentation. They are not documentation: they are a fixed budget spent on every request, enforced by a truncation cap that fails in the least visible way available. Put a number on the budget, guard the number mechanically, and treat "move it out of always-loaded" as the only change that actually buys anything.

## The Problem

**The cap truncates the middle, so the failure looks like nothing happened.** Typical implementations keep a head fraction and a tail fraction and drop what's between (Hermes: 70% head / 20% tail / marker in the middle 10%). The middle of a routing file is the routing table. An overflow therefore deletes *rules* while leaving the opening context and closing sections intact — the file reads fine, the agent's behavior quietly regresses, and the only signal is a line in an agent log. Overflow does not shave the footer; it removes the part you most needed.

**Headroom you bought once gets re-consumed silently.** A compression pass that frees 3,500 characters creates a dividend nobody is accounting for. Every subsequent well-justified addition spends a little of it. Nothing in the workflow reports the number, so the file drifts back to the edge over months and the next routine edit — a 460-character rule that is obviously worth adding — is the one that goes over. The compression work is real; without a guard its benefit has a half-life.

**"Just raise the cap" is available, sanctioned, and usually wrong.** Most harnesses expose the limit as config, and upstreams raise it for themselves (hermes-agent ships a ~70K `AGENTS.md` and suggests `context_file_max_chars: 80000`). But a bigger always-loaded file is a bigger bill on every turn *and* worse instruction-following. An ETH Zurich study (Feb 2026) found `AGENTS.md`-style context files **reduced** task success versus giving the agent no repository context at all, while adding >20% inference cost — human-written files improved results only ~4%, and on one model went negative. Separately, the probability of satisfying *every* instruction decays roughly exponentially with instruction count. Raising the cap removes the error message, not the problem.

**Relocating between two always-loaded files buys nothing but cap headroom.** The obvious fix for "`AGENTS.md` is full" is "move some sections to `SOUL.md`." Both are injected every turn. Total tokens: unchanged. Instruction density: unchanged. What you get is cap relief for one file — and you hand the *other* file the same cliff you just escaped. Worth doing with eyes open; worth never describing as a cost or quality win.

**Bytes are not characters.** Caps are typically in characters; `wc -c` reports bytes. A rules file dense with `—`, `→`, `✓`, and emoji runs 1–3% larger in bytes than characters, so a byte-based guard fires late — or a byte-based "we have room" reading is wrong in the dangerous direction.

## The Pattern

**1. Name the budget and print it where changes happen.** The cap is a number; put it in the deploy/commit path, not in a doc. Hard-fail over the limit, warn in the last few percent below it, and always state the remaining headroom so it is visible on a normal day rather than at the cliff:

```
ERROR:   AGENTS.md is 20179 chars — OVER the 20000 cap. Hermes cuts the middle out.
WARNING: AGENTS.md is 19979 chars — only 21 under the 20000 cap.
```

Count **characters** (`wc -m`), and apply it to every file the harness auto-loads, not just the one that bit you.

**2. Pin the cap explicitly, as a seatbelt — not as permission to grow.** Set the config value rather than riding a default floor, so the limit is a decision with a rationale instead of whatever the harness picks when an optional parameter is absent. Pin it modestly *above* your guard's ceiling: the guard is the policy, the pin ensures that if the policy is ever bypassed the failure is a visible refusal rather than a silent middle-ectomy. Before pinning, enumerate every file the change affects — a bigger cap applies to all of them.

**3. Verify against the harness's real loader, not the documented limit.** Caps are increasingly dynamic (scale with the model's window, floor when a parameter is absent, override from config), so the effective number depends on a call path you did not write. Load the actual file through the actual loader and assert on the result:

```python
out = _load_agents_md(Path(tmpdir))
print(len(out), "TRUNCATED" in out)
```

Two minutes of probing beat a documented constant. In the source incident the doc said 20,000, the dynamic path said 240,000 for a large-window model, and the truth for this loader was 20,000 — because that call site never threaded the window through.

**4. When it's full, move content OUT of always-loaded, not sideways.** The lever that reduces cost *and* instruction density is on-demand loading: skills the agent lists and opens when relevant, reference docs it reads when the topic arises, tool output it fetches instead of memorizing. What must stay always-loaded is the routing that tells it *when to go look* — a pointer is a line, the content it points to is a page.

**But this remedy is priced for a frontier model, and the price changes on a local one.** On-demand loading assumes two things that only hold at the top of the market: that a tool call is cheap, and that what you load lands in a window with room to spare. On a local model neither holds — the agent must spend a call to read the skill, and then carry it in a window where quality degrades far earlier than capacity runs out (practitioner testimony, 2026-08-11: usable behaviour falls off somewhere around 4–8K, against loadable ceilings many times that). So the same refactor that buys headroom on a frontier model can buy a *worse* agent locally: you converted a fixed cost into a per-use cost, on the runtime least able to pay it. **Route the decision by model class.** Where the worker is local, the honest options are fewer skills, smaller ones, or accepting the always-loaded cost — not "move it out and stop thinking about it." Where an agent's whole capability layer is on-demand skills, this stops being a tuning question and becomes an architectural one.

## Why It Works

The cap is not the enemy; invisibility is. Every failure above is silent by construction — a truncation warning in a log, a dividend with no ledger, a byte count that flatters. Attaching one number to the workflow converts all of them into an ordinary, early, boring signal. And the budget framing makes the right question automatic: not "is there room for this rule?" but "does this rule earn a place in every future request?"

## When to Use

- Any harness with auto-injected context files (`CLAUDE.md`, `AGENTS.md`, `SOUL.md`, `.cursorrules`, `.mdc` rules).
- Immediately after any compression/refactor that frees headroom — that is the moment the dividend needs a guard, not later.
- When a rules file is within ~10% of its cap, or when you cannot state its current size from memory.

## When NOT to Use

- Files read on demand (skills, reference docs) — **on a frontier model.** Their size costs nothing until used, and that is the whole point of moving content there. On a local model the exemption does not hold: the read is a tool call and the content lands in a scarce window, so an on-demand corpus needs its own budget rather than a pass. See the model-class caveat under point 4.
- Small, stable files nowhere near the limit. A guard on a 3K file is ceremony — though a cheap one costs nothing to leave running.

## Watch-outs

- **A warning in a log is not a signal.** If the only overflow evidence is a line in an agent log, assume nobody will ever see it. The guard has to live where changes are made.
- **Don't let the guard become the reason to raise the cap.** When it fires, the default response is to trim or relocate outward. Raising the limit is a deliberate, argued exception.
- **Check what else the pin touches.** Raising a shared cap silently changes every auto-loaded file's behavior; enumerate them first and confirm it is behavior-neutral today.
- **A rule that keeps bending is not a size problem.** If a behavioral rule needs to be in the always-loaded file because the agent ignores it elsewhere, that is `personality-as-discipline` territory — a tell plus a named incident, which is usually *shorter* than the prose it replaces.

## Adjacent Patterns

- `personality-as-discipline` — the cheaper shape for behavioral rules that bend; often removes bytes rather than adding them.
- `three-tier-memory-pipeline` — which tier a fact belongs in; this pattern is the budget constraint on tier one.
- `scheduled-llm-spend-gate` — the same "standing charge" framing applied to scheduled jobs instead of prompt bytes.
- `self-reporting-staleness-check` — the general shape of making silent drift announce itself.
- `verify-adoption-against-installed-source` — probe the installed loader rather than trusting documented behavior.

## Source

The household-agent repo, 2026-07-22. A Google re-auth rule was added to the household agent's `AGENTS.md`; the file went from 19,979 to 20,179 characters and a live loader probe showed it loading as 18,198 — the routing table's middle removed, with only an `agent.log` warning. Story 7-15 had compressed that same file two months earlier (19,999 → ~16,500) explicitly to create ~3,500 characters of headroom; all of it had been re-consumed unnoticed. Fixes: a char-based deploy guard (abort over cap, warn in the last 500), an explicit `context_file_max_chars` pin above that ceiling, and a deferred-work entry re-scoping the planned Phase 2 from *relocate to `SOUL.md`* to *reduce always-loaded content*, since both files load every turn.
