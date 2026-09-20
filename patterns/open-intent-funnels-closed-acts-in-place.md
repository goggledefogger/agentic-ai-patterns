---
type: pattern
date: "2026-08-17"
source: The dashboard, member-zero live dogfooding — one design rule sorted a day of UX feedback (dead-feeling clicks, invisible prefills, a universal edit gesture)
tags:
  - hci
  - agents
  - chat-ui
  - design
---

# Open Intent Funnels to Chat; Closed Operations Act in Place

An agent-backed UI keeps facing the same per-affordance question: should this button do the thing, or talk to the agent about the thing? Decided ad hoc, both failure modes ship: ceremony (a theme toggle that routes through a conversation) and dead ends (a status card that renders a fact and offers nowhere to go). One rule sorted every case in a day of live member feedback.

**The rule:** classify each affordance by its verb. A **closed** operation — toggle, pin, view, navigate, anything whose outcome is fully specified by the click — acts in place; routing it through chat is ceremony. **Open** intent — customize, ask about, set up, continue — is anything whose natural next step is a *sentence*: it prepopulates the chat composer with the thing's context and lets the user finish by typing or voice.

## The Pattern

- **Every informational surface is a doorway.** A file path, a connection tile, a status card, the mascot graphic: if it renders a fact, clicking it opens the conversation about that fact ("What can you actually do through X?", "About this note — what changed?"). A rendered fact with no doorway is a dead end.
- **The funnel must announce itself.** A silent prefill reads as a dead click when the eye isn't on the composer. One attention pulse on the input box + the guidance line stepping up (color, size) until typing starts. Reduced-motion keeps the color cue.
- **One universal edit gesture on top.** Right-click / long-press (Android fires `contextmenu` for long-press — one handler) on any region prefills an edit prompt for *that* region, from an ordered selector→name→prompt map walked with `closest()`. Text selections pass through to the native menu — people right-click to copy.
- **Never claim a knob that doesn't exist.** The chat can only change what has a real config surface. The honest ladder: do it; or name what's designed-but-coming plus today's workaround; or file the ask as a dated wish in the user's own notes and say so. Anything else is theater.
- **The knobs grow to meet the funnel.** Each time a behavior becomes config the agent can edit (a projects map file, an engine choice), a class of "not yet changeable" answers becomes real. Config-the-agent-can-write is what makes the gesture honest at scale.

## Why It Works

- The verb classification is checkable at design time — "is the next step a sentence?" has an answer, so new affordances stop being judgment calls.
- Doorways convert the UI from a report into an interface to the agent: the surfaces teach that everything routes back to the conversation, which is the product's actual capability.
- The announce-itself rule fixes the funnel's one failure mode (invisible success) without modal interruptions.

## When to Use

- Any UI whose backend is a conversational agent: dashboards over assistants, chat-adjacent panels, "AI-powered" settings surfaces.
- Retrofitting: sweep existing non-clickable renders (status cards, lists, graphics) and ask which are facts wanting doorways.

## Adjacent Patterns

- `the-agents-home-is-not-its-library.md` — same product, the architecture split under this UI rule.
- `personality-as-discipline.md` — the agent's voice carrying the answers these doorways open.
