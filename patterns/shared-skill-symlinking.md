---
type: pattern
date: "2026-05-13"
source: A hobby inventory vault
tags:
  - automation
  - tooling
  - architecture
---

# Shared Skill Symlinking (The Distributed Skill Layer)

Define AI agent skills (`SKILL.md` folders) directly inside the `.claude/skills` directory of the Obsidian vault they operate on, and then symlink those local skill directories into the global skill path (`~/.gemini/antigravity/skills/` or similar) so the skill is accessible globally across terminals without duplicating code.

## The Problem

When you build an AI agent skill that manages a specific domain—for example, a `/inventory` skill that tracks 3D printing filament in a specific Obsidian vault—the skill is tightly coupled to the schema and folder structure of that exact vault. 

If you put the skill in your global skills directory, it's separated from the codebase it governs. If the vault moves, the skill breaks. If you share the vault with someone else (via git), they don't get the skill. Conversely, if you *only* keep the skill inside the vault, you have to `cd` into the vault every time you want the global CLI agent to access it.

## The Pattern

Treat the Obsidian vault as the absolute single source of truth for the skill's logic, but use filesystem symlinking to inject the skill into your global toolchain.

### 1. Build inside the Vault
Create the skill locally:
```bash
/path/to/vault/.claude/skills/inventory/SKILL.md
```
Write the instructions, test the behavior, and iterate on it from within the vault. The skill stays version-controlled right alongside the markdown data it modifies.

### 2. Symlink to the Global Layer
In your terminal, create a symbolic link from the global skills directory pointing to the local vault skill:
```bash
ln -s /path/to/vault/.claude/skills/inventory ~/.gemini/antigravity/skills/inventory
```

### 3. Invoke Anywhere
You can now open a terminal *anywhere* on your computer—or inside completely unrelated repositories—and invoke `@[/inventory]`. The agent will load the symlinked `SKILL.md`, read the instructions (which contain absolute paths or context pointers back to the vault), and execute the task correctly.

## Why this works

- **No Drift:** You never have to manually sync a "global" version of the skill with the "local" version. There is only one file.
- **Portability:** When you commit and push the Obsidian vault to GitHub, the `.claude/skills/inventory` directory goes with it. Anyone who pulls the vault gets the skill automatically.
- **Contextual encapsulation:** A skill written to manage `Reference/Filaments/` belongs inside the vault where `Reference/Filaments/` exists. It makes logical sense for the schema definition to live next to the data.

## How to Adopt

1. Move any vault-specific global skills into the target vault's `.claude/skills/` directory.
2. Ensure the `SKILL.md` uses absolute paths or clear workspace resolution instructions if the agent might invoke the skill while its current working directory (`CWD`) is outside the vault.
3. Remove the old global copy.
4. Run `ln -s /path/to/vault/.claude/skills/your-skill /path/to/global/skills/your-skill`.
5. Test invocation from a totally unrelated directory (like `~/Desktop`).
