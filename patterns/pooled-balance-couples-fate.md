---
type: pattern
date: "2026-07-20"
source: The personal agent session (2026-07-20) migrating the household agent's Gemini API projects onto the personal agent's prepaid billing account to share its credit balance; a "make sure we're doing this well" web check surfaced that pooling every project on one prepay balance couples their fate, fixed with per-project spend caps.
tags:
  - architecture
  - billing
  - cost
  - resource-management
  - isolation
---

# A Pooled Balance Couples Fate — Cap Each Consumer

Consolidating several consumers onto one shared prepaid balance (a billing account's credits, a quota, a rate-limit tier, a shared budget) is the obvious way to let them all draw on a pool you funded once. The catch that isn't obvious until it bites: pooling **couples their fate**. When the pool hits zero, *every* consumer stops at once, and any single greedy consumer can drain the pool the others depend on. Get the sharing you wanted, then fence it with a per-consumer cap.

## The Problem

You have credits (or quota, or budget) on one account and several projects that should run on it. You link them all to the shared pool and it works — usage draws the shared balance in real time, exactly as intended. But you've quietly created two failure modes:

- **Shared depletion.** When the pooled balance reaches zero, *all* consumers linked to it stop simultaneously. A runaway in a low-priority project takes down the high-priority one that happened to share the account.
- **No blast-radius containment.** The whole reason a consumer was isolated (its own project, its own key) is undone at the billing layer — one bad loop now spends the entire pool, not its own slice.

The move reads as "better separation" (one funded account, clean ownership) while actually *reducing* isolation on the axis that matters under failure: spend.

## The Pattern

**Pool the balance for convenience, but cap each consumer so one can't starve the others — and verify where the pooled value actually lives before you migrate onto it.**

1. **Verify the pool before pointing anything at it.** A claim like "that account has credits" is a premise, not a fact. Confirm the balance on the surface that actually holds it — pooled value often shows on one console and reads as empty on another (prepaid Gemini credits show in AI Studio's billing view but *not* on the Cloud Console "Credits" page, which lists only promotional credits). Don't re-point a consumer onto a pool you haven't seen with your own eyes.
2. **Link the consumers** to the shared pool (the sharing you actually wanted).
3. **Set a per-consumer hard cap** sized to that consumer's real footprint plus headroom — not a share of the whole pool. The cap is what converts "shared fate" back into "bounded fate": a runaway hits its own ceiling long before it drains the pool.
4. **Prefer graceful-stop over auto-refill** for the pool as a whole, so depletion fails safe (requests stop) rather than silently charging a backing payment method.

## Why It Works

- **The cap restores the isolation the pool removed.** Consumers share the *funding* but not the *failure* — each can only spend up to its ceiling, so the high-priority consumer survives a low-priority runaway
- **Fail-safe depletion beats surprise spend.** A hard stop at zero is a visible, recoverable event; auto-refill turns a runaway into an invoice
- **Verifying the pool first prevents the confident wrong move** — re-pointing a live consumer onto an "empty-looking" or non-existent balance is worse than leaving it where it was

## When to Use

- Migrating multiple projects/services onto one billing account, credit pool, or quota to share a balance
- Any shared prepaid or capped resource (credits, tokens, connection pool, shared monthly budget) gaining a new consumer
- Whenever "consolidate for shared credits" is proposed as "better separation" — it improves ownership separation and *worsens* spend isolation unless capped

## When NOT to Use

- A single consumer on its own account — nothing to starve, a cap is just an extra knob
- Pools where per-consumer caps aren't offered and the platform already isolates spend per consumer another way (then rely on that, but confirm it exists)

## Watch-outs

- **"It shows zero" may be the wrong console, not an empty pool.** Check the surface that actually owns the balance before concluding anything
- **Size caps to the consumer, not the pool.** A cap set to the full pool balance provides no isolation — it only stops the account tier cap from being the sole guard
- **Caps can lag.** Enforcement may be eventual (e.g. a ~10-minute latency window with possible overage). Treat the cap as a strong fence, not an instantaneous circuit breaker
- **Coupling is the price of sharing.** If two consumers must *never* share fate, they need separate pools, not one pool with caps — caps bound the damage, they don't fully decouple

## Adjacent Patterns

- `router-worker-exfil-containment.md` — the other "the isolation you assumed isn't there unless enforced" shape, applied to egress instead of spend
- `scan-prior-art-before-building-infra.md` — the "make sure we're doing this well" scan that surfaced the shared-fate risk here before it shipped
- `registry-based-monitoring.md` — where the pool and its consumers get a pointer row so the coupling is visible later
- `delegation-decision.md` — deciding whether a consumer deserves its own isolated resource at all

## Source

The personal agent session, 2026-07-20. The household agent's Gemini API projects had exhausted their own billing account's credits, while a separate account ("the personal agent AI Billing") held a prepaid balance. The projects were re-linked to the personal agent account so their usage would draw its credits — correct, and confirmed against Google's docs: prepaid credits apply at the billing-account level, so any linked project draws them. The same docs surfaced the catch: *"when your Prepay credit balance on the billing account hits $0, all API keys in all projects linked to that billing account will stop working simultaneously."* Consolidating for shared credits had coupled the fate of three projects — including the operator's own — onto one balance. Mitigation: per-project monthly spend caps (set in AI Studio, a hard stop distinct from the account tier cap) sized to each project's small real footprint, so a runaway in one can't starve the others. A prior detour also seeded watch-out #1: the credit balance read as empty on the Cloud Console "Credits" page while sitting plainly in AI Studio's billing view — same account, different surface.
