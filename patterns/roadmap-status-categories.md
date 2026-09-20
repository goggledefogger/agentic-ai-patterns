---
type: pattern
date: "2026-04-18"
source: A course participant's home-agent system (the workspace repo)
tags:
  - documentation
  - roadmap
  - living-doc
---

# ROADMAP.md Status Categories

A ROADMAP.md organized not by priority or quarter, but by **epistemic state**: what you trust, what you don't, what's in flight, what's on the shelf. Each item carries its own `Decided:` and `Open question:` notes inline, so the rationale lives next to the thing.

## The Shape

Five categories, always in this order:

```markdown
## Works
Reliable. Don't re-solve these.

### Interaction capture (`interaction-capture`)
- Gmail, iMessage, WhatsApp, Calendar → Airtable CRM every 15 minutes
- **Decided**: Deterministic Python, not LLM-powered. Consistency matters more than intelligence here.

## Works but fragile
Functional but has known issues. Be careful here.

### Post-meeting micro briefings
- 7am daily, Granola-enriched Slack cards for yesterday's meetings
- **Fragile because**: Depends on granola-export having run recently. If the Mac slept overnight, the cache may be stale.

## Half-built
Partially implemented. Picking up here is cheap; starting fresh is expensive.

## Next
What's actually teed up. Not wishlist — stuff with a path to done.
- **Open question**: Cadence? On-demand vs scheduled?

## Someday
Ideas worth remembering. Zero commitment.
```

## Why It Works

- **Works vs Works-but-fragile**, separates "reliable" from "functional but don't poke it." Prevents the fragile one from being treated as stable in new work
- **Decided:** markers next to each item capture rationale at the level where the decision applies, not in a separate doc that drifts
- **Open questions:** inline keep uncertainty visible. Claude reads these and flags them back in new sessions
- **Next is small. Someday is where wishes live.** Keeps Next honest, if something sits there for weeks without motion, it belongs in Someday
- **Refreshes cleanly** because each section has a bounded scope. Updating the "Works but fragile" section is a local edit

## When to Use

Any project with 5+ moving parts where you keep losing track of what's stable vs. in flight. Especially good when Claude is reading the roadmap to decide what to touch, the categories tell it what's safe vs. risky to change.

## Source

`~/src/participant-system/ROADMAP.md`. The canonical example, every category is populated with real the participant's system agents, each tagged with Decided / Open question / Fragile because. Worth reading end-to-end as a model.
