---
type: pattern
date: "2026-08-29"
source: Astrolabe / the dashboard — the make-it-yours funnel, the scene library, the model-lane registry; the author's framing 2026-08-29 and the "software factory" coinage at working session #16 (the course repo, issue #173)
tags:
  - architecture
  - skills
  - extensibility
  - safety
  - product
---

# A Free-Form Ask Lands on Decided Rails

The moment a product lets its members ask an agent to change the product itself — "make the background warmer", "add me a copy button", "build me a website" — every such ask becomes an engineering decision made by whoever answers it. If that is a model with a blank canvas, the member's innocent sentence can import a framework nobody chose, wire a dependency nobody will maintain, or quietly break the foundation they cannot see. The member did nothing wrong. The product left the decision to whatever the model reached for, and the same ask produces a working feature one day and a mess the next. Structure or luck, and they cannot tell which until after.

The fix is not to narrow what members may ask. It is to have already made the decisions their asks will need, and to ship those decisions as rails the agent lands on — invisible until summoned.

## The Pattern

**1. Decide the engineering in advance; the ask only picks within it.** The rail names which files may be written, which vocabulary is available, which frameworks exist (often: the ones already shipped, and no others). Astrolabe's make-it-yours skill is the worked example: a member describes any look they want, and the agent may write exactly two overlay files in the member's own folder — never the app's — preferring token overrides so one change moves the whole room coherently. The member's freedom is real; the blast radius was chosen years before their sentence.

**2. Every rail carries a "what never enters" clause.** Positive instructions ("write this file, prefer these tokens") leave the negative space to the model, and the negative space is where the damage lives: `@import` of a CDN stylesheet, an external `url()` that makes a member's look phone home, `display:none` on a load-bearing control. Name the prohibitions in the rail itself, and ideally back them with mechanical enforcement (e.g. AST validation, strict JSON schema bounds) since models can hallucinate past prompt instructions. A rail without its never-clause is a suggestion.

**3. Extension points take declarations, not code.** When members (or their agents) can add behavior, the seam accepts *names over a bounded vocabulary*, never executable material. Astrolabe's scene library states it exactly: names, not functions, cross the door — an unknown name does nothing and says so; a member-authored overlay would declare scenes over the same vocabulary, one file to delete to undo. Same shape in its model system: a lane is a model list plus an env delta behind a small contract; a member's own model file merges by id. The vocabulary is the guardrail.

**4. Rails load lazily and stay invisible until needed.** The cost of pre-made structure must not be paid by the member who never asks. Ship skills dormant beside the product (read at ask time, so they update without redeploying the conversation), reveal capability progressively as it is earned, and let discovery of heavier machinery be member-initiated rather than ambient. There can be far more behind the scenes than is ever visible early — that asymmetry is the point, not a smell.

**5. When an ask outruns the rails, the rail says so honestly.** The funnel's last rungs: map the ask to a knob that exists and turn it; name a knob that is designed but unbuilt, with today's workaround; or record the wish, dated, in the member's own notes — their asks are design input. What the rail never does is improvise infrastructure to avoid saying "not yet".

## The Tension It Resolves

This pattern is how a power user pushing extensibility and a product owner protecting primary experiences stop fighting: the pushing happens by *adding rails* (a new skill, a new vocabulary entry, a new lane behind the contract), never by loosening the ones the main path stands on. The default experience stays untouched by construction — the Claude-only member sees the same flat list; the member who never customizes never loads the customize skill — while the frontier moves as fast as new rails can be written.

## When It Shows Up

Any agent-mediated product where "users can ask for changes" is a feature: UI customization funnels, plugin/marketplace systems, "build me X" trajectories, model/provider pickers. The tell that the pattern is missing: a support burden shaped like "it worked when they asked, then broke", and diffs from member asks that no maintainer can predict the shape of.

## Related

- [[a-model-never-picks-the-destination]] — the same split one level down: models fill in content, deterministic structure decides where it lands.
- [[quick-win-scaffolding]] — rails at bootstrap time: minimal core now, mature patterns named as opt-in seeds.
- [[policy-on-every-door]] — a rail wired to one entry path silently skips on the others; attach it to the activity class.
