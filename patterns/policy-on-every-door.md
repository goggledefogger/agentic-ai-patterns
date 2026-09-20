---
type: pattern
date: "2026-07-28"
source: The personal agent 2026-07-28 session — the patterns-consult reflex skipped when planning entered through BMAD instead of the self-improve skill
tags:
  - process
  - skills
  - anti-pattern
---

# Policy on Every Door: A Standing Rule Wired to One Entry Path Silently Skips on the Others

A router had a standing rule: consult the patterns library before any impactful self-change. The rule lived in two places — step 1 of its self-improvement skill, and a memory line. Then a planning session entered through a *different* skill (a BMAD epics workflow), designed a safety-adjacent hook three steps deep, and the consult never fired. When it finally ran — because the human asked — it materially contradicted the design. The rule was real, documented twice, and attached to the wrong thing: an entry door, not the activity.

The failure is structural, not mnemonic. A policy attached to one skill fires only when work arrives through that skill. Work arrives through whatever door is nearest — a different framework, a plain conversational pivot, a resumed session — and each of those doors skips the rule without anyone deciding to skip it.

## The Pattern

1. **Attach policies to the activity class, not the entry skill.** "Before proposing a design change" is the trigger; "when the improve-X skill runs" is one door among several.
2. **Use each framework's own load-path mechanism.** Most skill/workflow systems have a pre-execution surface (activation steps, prepend hooks, session-start config). Wire the policy there, in *every* framework that can host the activity — the personal agent fix was one overlay file per BMAD planning skill (`activation_steps_prepend`), so the consult now runs before any epic or story structure is proposed, regardless of who remembers.
3. **A memory line is a bookmark, not a control.** If the policy matters, the test is: delete the memory — does the policy still fire? If not, it was never wired.

## When It Shows Up

Any system with more than one way to start the same kind of work: multiple planning skills, a CLI and a chat surface, resumed vs fresh sessions. The tell is a rule that everyone agrees exists, that has fired successfully before, and that is absent from the transcript of the session that needed it — its past firings were all through the one door it was nailed to.
