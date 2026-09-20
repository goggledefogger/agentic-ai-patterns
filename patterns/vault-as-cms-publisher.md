---
type: pattern
date: "2026-05-13"
source: A hobby inventory vault publishing a public collection site
tags:
  - architecture
  - automation
  - publishing
---

# Vault-as-CMS for Static Site Publishing

Use a private Obsidian vault as a headless CMS for managing state, physical inventory, or content, and use a deterministic Python script to parse the markdown (frontmatter and body) to generate a public static site on GitHub Pages.

## The Problem

When you have a collection of physical items (like 3D printing filaments, a book library, or an inventory of parts) that you want to share publicly, maintaining a database or a complex web app is overkill. However, manually updating an HTML file or a static site generator's JSON every time you use 50g of filament creates friction. If the friction to update the public gallery is too high, it falls out of date.

## The Pattern

Keep your single source of truth entirely within your private Obsidian vault using standard markdown and frontmatter. Use a Tier 1 script to parse the vault, perform necessary arithmetic or transformations, and push the compiled result to a separate public repository.

### 1. The Obsidian Vault (The Database)

Each item gets its own `.md` file in the vault. 
- **Frontmatter** stores the static attributes (brand, material, color, total weight, image filename).
- **Body Content** stores the qualitative, human-readable log (e.g., a "Usage Log" section where you append bullet points of what you printed and roughly how much material was used).

### 2. The Publisher Script (The ETL Pipeline)

A deterministic Python script (`scripts/publish-gallery.py`) acts as the bridge. It does three things:
1. **Parses:** Reads the frontmatter from all relevant notes.
2. **Calculates:** Uses regex to parse the human-written "Usage Log" bullet points (e.g., "approx 30g used") to mathematically calculate remaining weight or derived statuses.
3. **Generates & Pushes:** Compiles this into a `data.json` or renders an `index.html` directly, copies the associated image assets, and pushes the result to the public repository.

### 3. The Public Repo (The Frontend)

A separate repository (e.g., `user/filament-gallery`) hosts the static assets on GitHub Pages. It has no backend, no database, and no markdown parsing logic—it just renders the JSON or HTML provided by the publisher script.

## Examples to learn from

**Success: A public collection site.** the author's a hobby inventory vault tracks over 20 filament spools. Each spool has a markdown note. When the author prints something, he adds a bullet point to the Usage Log. A Python script (`publish-filament-gallery.py`) uses regex to parse the grams used, subtracts it from the frontmatter's total weight, flags recycled materials, copies the images, regenerates the static `index.html`, and pushes to the `filament-gallery` GitHub Pages repo.

## Why this works

- **Zero-friction data entry:** The human interface is just typing markdown in Obsidian. No web forms, no CMS logins.
- **Arithmetic from prose:** By using regex to extract numbers from natural language logs (`- [[Project]] — approx 30g used`), you get the benefits of quantitative tracking without losing the qualitative journaling.
- **Strict boundary:** The private vault remains private. The public repo only receives the sanitized, compiled output. 
- **Resilience:** If the publisher script breaks, the source of truth (the vault) is untouched. If the static site goes down, you can regenerate it instantly.

## How to Adopt

1. Define the frontmatter schema for your items in Obsidian.
2. Decide on a standardized phrasing for your log entries if you need mathematical extraction (e.g., "approx Xg used").
3. Write a Python script using `frontmatter` and `re` to parse the vault directory.
4. Have the script output a static `.json` or `.html` to a local clone of your public GitHub Pages repo.
5. Wire the script to a Git hook (like `.git/hooks/post-commit`) to run in the background (`python3 scripts/publish-gallery.py &`). This decouples the sync completely from human or LLM memory, ensuring it fires automatically every time the vault state is committed.
