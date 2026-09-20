---
type: pattern
date: "2026-08-17"
source: The dashboard member-zero test, 2026-08-13 → 2026-08-17 — the "which folder is my brain?" question recurring three times before the variable was split
tags:
  - architecture
  - memory
  - agents
  - separation-of-concerns
---

# The Agent's Home Is Not Its Library

A dashboard fronting a "second brain" keyed everything on one directory: the chat session spawned there, the file watcher watched there, unsaved-work counts, backup status, and the projects list all read there. For its designed user — one person, one vault — the fusion is invisible and correct. The first integrated user (multiple vaults with an agent layer routing across them) hit the same question three times in four days: *point it at the agent, and the visuals go dark, because the agent's folder is a routing layer built to stay nearly empty; point it at one vault, and the surface misses most of my work.* Neither answer was wrong. The variable was — one directory was wearing two jobs.

A mature personal-AI system decomposes into organs: the **library** (the memory stores — vaults, notes), the **librarian** (the agent that acts and routes across them), and often a **voice** (the output layer owning register and medium — retuning the same agent per channel). A young system fuses all of these into one folder, and tools built against young systems inherit the fusion silently. The failure only appears at the moment of success: the user whose system grew is exactly the user the tool loses.

## The Pattern

- **Name the two concerns as two variables, even while they hold the same value.** The *home* — where the process runs, where identity and state live, who you talk to. The *roots* — the set of stores the tool observes and reports on. Default the roots to `[home]` so the fused case is unchanged byte for byte.
- **Roots are a set, never a second scalar.** The moment the second concern exists it is plural (a `PATH`-style env var, a config list). A `SECONDARY_DIR` merely re-fuses at n=2.
- **Aggregations must be honest about the set.** Counts sum; a "backed up" headline reports the *stalest* store's last push, never the freshest — "backed up" must not mean "one of them is." Labels travel with items so two stores holding the same name don't conflate.
- **The topology is the user's local profile, not the tool's config.** The engine stays generic; which folders one person's brain spans lives in their launch environment (see `portable-engine-local-profile.md`).
- **Actions lag observation, explicitly.** Watching n stores is cheap; *acting* on n stores (save where? push what?) is the genuinely hard part. Ship multi-root observation with single-root actions and say so, rather than silently promising symmetric behavior.
- **Let identity anticipate the split.** If the tool reads a `SOUL.md`-style identity file, shape it so name (identity), agency (the librarian), and voice are distinct sections. Costs nothing at birth, gives the organs somewhere to grow.

## Why It Works

- The fused variable is a ceiling that never announces itself — every single-store user validates the design daily, and the tool has no error path for "my system has layers." Splitting home from roots removes the ceiling without a migration: the default *is* the old behavior.
- "The member's instinct is the spec": a question the same user asks three times despite receiving the correct rule each time isn't a documentation gap, it's the design telling you which variable to split.
- Observation-before-action keeps the change small enough to ship the week the need lands, and the aggregate views immediately produce findings the single-root view structurally could not (a store with unpushed commits nobody was looking at).

## When to Use

- Any tool keyed on "the project directory" that both *runs somewhere* and *reports on somewhere*: dashboards, status lines, backup monitors, session managers, file watchers.
- When users start asking to point the tool at an orchestrating layer (an agent home, a workspace root, a meta-repo) and the answer has been "that breaks the views."
- Designing for two user archetypes where one is the other's future: the simple default must be the fused case, and the split must cost the simple user nothing.

## Adjacent Patterns

- `portable-engine-local-profile.md` — where the roots list lives: the user's environment, never the shipped engine.
- `memory-substrate-selection.md` — chooses what each store *is*; this pattern is about a tool observing several of them at once.
- `identity-is-a-property-of-the-set.md` — same root failure at a different altitude: a per-instance design silently breaking at the set level.
