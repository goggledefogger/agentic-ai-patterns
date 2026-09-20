---
type: pattern
date: "2026-08-20"
source: The dashboard — turning the import-a-skill flow and a new remote-control capability into pinnable "rituals," where the durable, shareable part is a skill and the per-surface part is a thin card
tags:
  - architecture
  - skills
  - portability
  - agent-ui
---

# A Ritual Is a Skill Plus a Wrapper

An agent-backed UI accumulates named actions — "rituals," "routines,"
"commands," pinnable buttons. The tempting shape is to define each as UI: a
label, an icon, and a prompt string the button sends. That fuses two things
that want to live apart: the **capability** (what the action actually does,
which is worth sharing and works on more than one platform) and the
**wrapper** (how *this* surface invokes it and what invoking it entails here).

Split them. The capability is a **skill** — a `SKILL.md` with frontmatter,
readable and runnable anywhere skills are (the same file a sibling platform
loads). The wrapper is a small record: a name, one plain line, the phrase the
button types (or the closed op it toggles), and a `skill:` field naming the
capability it invokes. Built-in rituals and member-imported ones are the same
shape on two different shelves.

```json
{ "id": "remote-control", "label": "Remote control",
  "skill": "remote-control", "desc": "Reach your brain from your phone or the web." }
```

## Why the split pays

- **Sharing.** A ritual that is a skill can be sent to someone, PR'd, or
  carried to another agent — the wrapper is throwaway, the skill is the value.
  A ritual that is only a prompt string is trapped in the UI that defined it.
- **Cross-platform without forking the behavior.** One skill can detect where
  it runs and take the right door — and say honestly where a platform has no
  equivalent — instead of the host reimplementing the capability per surface.
  (A remote-control skill: dashboard door, terminal door, "no equivalent yet"
  for the third; see [[open-intent-funnels-closed-acts-in-place]] for why the
  honest "not here" beats a faked knob.)
- **The maker becomes a skill too.** The thing that *turns a skill into a
  ritual* is itself a skill (`skill-to-ritual`): read the skill, say what it
  does in plain words, get consent, install the wrapper, never run on import.
  So the import flow is shareable and testable, not prose only one prompt
  knows — the same reason [[shared-skill-symlinking]] keeps skills as files.

## The discipline

**Skill first, card second — never a card only a prompt knows how to honor.**
If you find yourself writing a button whose behavior lives entirely in a
system-prompt sentence, you have a wrapper with no skill under it: the action
can't be shared, tested, or carried elsewhere. Write the `SKILL.md`, then wrap
it.

Keep the wrapper thin. Its job is to *invoke and locate*, not to re-specify —
label, line, phrase-or-toggle, and the skill name. Everything about what the
action does belongs in the skill. A wrapper that grows conditionals is
capability leaking back into the UI.

## Related

- [[shared-skill-symlinking]] — skills as files in the vault, invocable globally
- [[portable-engine-local-profile]] — the same split one level down: portable
  engine + machine-local profile
- [[open-intent-funnels-closed-acts-in-place]] — which wrapper a ritual gets
  (a phrase-typing funnel vs a toggle) and the never-fake-a-knob rule
- [[a-capability-contract-must-name-the-verb]] — the wrapper's verb is the contract
