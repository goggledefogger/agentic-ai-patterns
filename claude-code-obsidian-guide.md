---
type: reference
date: "2026-03-09"
tags:
  - meta
  - claude-code
  - obsidian
---

# Claude Code + Obsidian Guide

> **Last refreshed: 2026-07-02.** Verified against current recommendations as of this date. Deltas live in git history, not here (see `patterns/living-doc-refresh-ritual.md`).

How to set up and run an Obsidian vault with Claude Code as your working memory. Written for a fresh Claude Code session inheriting an existing vault or bootstrapping a new one.

It names a few helper scripts in passing (`theme-vault.sh`, `workspace-identity.py`, a repo digest). Those live in a private companion repo and are not shipped here; each section describes the mechanism well enough to rebuild in a few lines.

## State of recommendations

Current posture on the bigger calls the guide makes. Lightly opinionated, epistemic-state-aware. Update this section when a recommendation changes materially, not when a section reorders.

**Works, do this by default:**

- kepano/obsidian-skills as the foundational skill layer for every vault
- Bases over Dataview for first-party structured views that stay portable
- Daily notes as session logs, not an inbox
- Per-vault Catppuccin theming so you always know which vault you're in
- obsidian-git wired up for ambient backup + multi-device sync
- Priority hierarchy (scripts > obsidian markdown > skills > docs > memory) as the default tie-breaker when there's more than one right place for a thing

**Works but with caveats:**

- Multi-repo observatory pattern, solid once set up but ROADMAP maintenance across symlinked repos is still manual
- hotkey-passthrough plugin, useful but young. Expect occasional compat quirks on Obsidian updates
- Local audio transcription, fast and accurate on Apple Silicon with mlx-whisper but cross-platform parity requires faster-whisper with its own tradeoffs

**Next to try / half-baked:**

- Living-doc refresh ritual on guide-style docs (this very file is the first test case)
- Decision-doc ADRs for meta choices in the vault (pattern documented in `patterns/decision-doc-adr.md`, adoption still light)
- Filesystem-queue pattern (`patterns/filesystem-queue.md`) for sensor→processor flows in vault automation, not yet exercised end-to-end in a real vault

## The Priority Hierarchy

This governs every decision. Memorize it.

1. **Scripts/code**, deterministic, version-controlled, repeatable
2. **Obsidian markdown**, proper format with frontmatter + wikilinks
3. **Skill.md skills**, reusable Claude Code skills
4. **Docs**, documentation files
5. **memory.md**, last resort only

If something can be a script, it's a script. If it can be a note, it's a note. Memory is for things that don't fit anywhere else.

## Foundational Skill: kepano/obsidian-skills

Install [kepano/obsidian-skills](https://github.com/kepano/obsidian-skills) into every vault. This is the official skill set from Steph Ango (Obsidian's CEO) that teaches Claude the correct formats for all Obsidian-native files. Five skills:

| Skill | What it teaches |
|-------|----------------|
| **obsidian-markdown** | Obsidian Flavored Markdown — wikilinks, embeds, callouts, properties |
| **obsidian-bases** | `.base` files — Obsidian's database format (views, filters, formulas) |
| **json-canvas** | `.canvas` files — JSON Canvas spec (nodes, edges, groups) |
| **obsidian-cli** | Interact with vaults via the Obsidian CLI (plugin/theme dev) |
| **defuddle** | Extract clean markdown from web pages using Defuddle (saves tokens) |

### Installation

**Preferred. Claude Code plugin install:**
```
/plugin marketplace add kepano/obsidian-skills
/plugin install obsidian@obsidian-skills
```

**Manual, clone into vault:**
```bash
cd /path/to/vault
git clone https://github.com/kepano/obsidian-skills /tmp/obsidian-skills
cp -R /tmp/obsidian-skills/skills/* .claude/skills/
rm -rf /tmp/obsidian-skills
```

> **Note:** The repo nests skills under a `skills/` subdirectory, don't clone the repo root directly into `.claude/skills/` or you'll get the wrong structure.

**Tier:** Skill (tier 3 in the priority hierarchy), correct tier, and foundational for everything else Claude does in a vault. Install this before writing any Obsidian files.

## First Session Checklist

### 1. Read before you do anything
- Read `CLAUDE.md` at vault root, this is the contract
- Read `README.md` for structure overview
- Scan folder structure with `ls` to understand what exists
- Check `.claude/` for existing skills and commands (including obsidian-skills)

### 2. Understand the vault structure
Every vault follows this pattern:
```
Daily/              → YYYY-MM-DD session notes
Meetings/           → meeting notes
Projects/           → subfolders, each with CLAUDE.md + files.md
People/             → one file per person
Reference/          → technical docs, decisions, guides
Templates/          → Obsidian templates
Reports/            → versioned analysis reports (insights, audits, security)
Inbox/              → human-facing drafts, delete after sending
Assets/             → attachments
sources/transcripts → processed transcript outlines
```

### 3. Bootstrap a new vault (if starting fresh)

If there's no existing vault (just a default `Welcome.md`), create the full structure:

```bash
cd /path/to/vault
mkdir -p Daily Meetings Projects People Reference Templates Inbox Assets sources/transcripts .claude/skills
```

**Obsidian config files**, write these to `.obsidian/` so core plugins are pre-configured:

```json
// daily-notes.json
{ "folder": "Daily", "template": "Templates/Daily Note" }

// templates.json
{ "folder": "Templates" }

// app.json — routes stray new files to Inbox
{ "newFileLocation": "folder", "newFileFolderPath": "Inbox" }
```

**Starter `.gitignore`:**
```
.obsidian/workspace.json
.obsidian/plugins/
.DS_Store
.claude/
```

Add any symlinked directories to `.gitignore` too (see "Symlinked External Repos" below).

**Bases views**, create `.base` files at the vault root so Obsidian auto-populates dashboard tables from frontmatter:

```yaml
# Projects.base
filters:
  and:
    - file.hasTag("project")
    - 'file.ext == "md"'
properties:
  status:
    displayName: Status
views:
  - type: table
    name: Active Projects
```

```yaml
# People.base
filters:
  and:
    - file.hasTag("person")
    - 'file.ext == "md"'
properties:
  role:
    displayName: Role
views:
  - type: table
    name: People
```

**Home.md**, create a dashboard that embeds the Bases views and links to key folders:

```markdown
---
type: home
tags:
  - dashboard
---

# Vault Name

## Quick Links
- [[Daily/|Daily Notes]]
- [[Meetings/|Meetings]]
- [[Projects/|Projects]]
- [[People/|People]]
- [[Reference/|Reference]]
- [[Reports/|Reports]]
- [[Inbox/|Inbox / Drafts]]

## Active Projects
![[Projects.base#Active Projects]]

## People
![[People.base]]

## TODOs
- [ ] ...
```

**Install obsidian-git**, copy from an existing vault or install via Obsidian's community plugin browser. Add `"obsidian-git"` to `.obsidian/community-plugins.json`. Initialize git, create a private GitHub repo, and push:
```bash
cd /path/to/vault
git init
git remote add origin https://github.com/user/vault-repo.git
git add . && git commit -m "Initial vault setup"
git push -u origin master
```

### 4. Check Git and GitHub
- Confirm which GitHub account is active: `gh auth status`
- Keep one account for personal projects and one for work, and know which is which
- If the wrong account is active, surface the mismatch and let the user switch it. Never run `gh auth switch` or `gh auth login` from a session unless explicitly asked, account state belongs to the user
- Check remote: `git remote -v`

## The Non-Negotiable Rules

### Wikilinks everywhere
Every person, project, or concept that has a file gets linked every time it's mentioned:
```markdown
Talked to [[jane-smith|Jane Smith]] about [[Projects/api-migration|API Migration]].
```
Never use plain text for linkable entities. This powers graph view, backlinks, and navigation.

### Frontmatter on every file
```yaml
---
type: daily | meeting | project | person | reference | draft
date: "YYYY-MM-DD"
tags:
  - lowercase-hyphenated
---
```
Additional properties vary by type (status, attendees, role, etc.). Check `Templates/` for the canonical shapes.

### Tags in frontmatter only
Never inline tags. Always lowercase with hyphens. Tags are for cross-cutting themes that don't deserve their own file.

### One source of truth
Content lives in ONE place. Every other mention links to it. If you find yourself copying content, stop and link instead.

### Propagate updates
When any fact changes, a person's role, a project status, a decision, search the entire vault and update every reference. Use `grep` mentally: where else is this mentioned?

## Input Modes

Most vault owners work in two modes:

### Quick drop (default)
Short input → identify the right file → route it there → confirm → done. Don't over-process. Don't ask clarifying questions unless genuinely ambiguous. Most inputs are quick drops.

### Deep dive
The user will signal when something needs discussion, multi-file work, or extended drafting. Don't assume deep dive.

### Daily notes are session logs, not an inbox
Daily notes record what happened and what was discussed. Route actionable content to its destination file immediately, don't leave it in the daily note for later triage. If something belongs in a person file, project CLAUDE.md, or reference doc, put it there now and link back from the daily note if needed.

## Drafts Workflow

- Write drafts to `Inbox/` so they're visible in Obsidian
- Always include subject lines for emails
- Delete draft files after the user confirms sent/done
- Sensitive content goes to `/tmp/` (outside the vault/repo)

## Project Subfolders

When a project is substantial enough for its own folder:

```
Projects/project-name/
├── CLAUDE.md      → context, status, active TODOs
├── files.md       → file inventory with source URLs, dates, freshness
├── notes.md       → running notes
└── ...            → other project-specific files
```

`CLAUDE.md` = "what's going on." `files.md` = "what's here and how fresh."

### ROADMAP.md for complex systems

For vaults that track evolving multi-component systems, add a `ROADMAP.md` that captures uncertainty and decisions, not just current state. Inspired by that system's ROADMAP (the private org repo is no longer public). The template is a living architecture doc: status categories plus inline `Decided:` and `Open question:` markers.

The key structure, five status tiers:

```markdown
## Works              — Reliable. Don't re-solve these.
## Works but fragile  — Functional, known issues. Be careful.
## Half-built         — In progress. Design decisions noted so Claude doesn't re-litigate.
## Next               — What to build next, with enough context to start.
## Someday            — Ideas, not commitments.
```

Each item uses **Decided** markers for settled questions ("don't re-litigate these") and **Open questions** for unsettled ones ("discuss freely"). This lets a new Claude session immediately know what's locked vs what's up for debate.

## Session Hygiene

Before ending any session with meaningful changes:
1. Update relevant project `CLAUDE.md` files (status, completed TODOs)
2. Update `files.md` if new files were added
3. Update root `README.md` if vault structure changed
4. Commit with a clear message and push

## How to Process Inputs

### Meeting transcript
1. Create processed outline at `sources/transcripts/YYYY-MM-DD-topic.md`
2. Add frontmatter with date, attendees, source info
3. Route actionable content to every relevant file (people, projects, etc.)
4. Wikilink all people mentioned
5. Don't save raw transcripts, only the processed outline

### Person update
1. Read their file in `People/` first
2. Update the person file
3. Search vault for all mentions, update related files
4. If new person, create from `Templates/Person.md`

### Project update
1. Read the project `CLAUDE.md` first
2. Update status, TODOs, notes
3. Propagate to related people files, meeting notes, daily notes

### Quick note / info drop
1. Identify where it belongs (which file, which section)
2. Route it there directly
3. Add wikilinks to any referenced people/projects
4. Confirm placement with the user

## Context First — Always

Never write blind. Before editing any file, read it. Before writing about a person, read their file. Before touching a project, read its CLAUDE.md. The vault is the source of truth, not your assumptions.

## Plugin Philosophy: Stay Lean

Default to minimal plugins. Every plugin is a dependency, it adds maintenance, can break on updates, and may lock you into Obsidian-specific patterns. Only add plugins when built-in features genuinely can't do the job.

### Bases over Dataview

Use Obsidian Bases (`.base` files) instead of the Dataview community plugin for dashboard views and queries.

- **Bases**: First-party, no plugin dependency, clean YAML syntax, the obsidian-skills kit already teaches Claude how to write them
- **Dataview**: More mature and flexible (inline queries inside notes), but it's a plugin dependency and embeds queries inside your content files
- **Portability**: Neither renders outside Obsidian. But Bases are separate YAML files, your actual content stays in pure markdown with standard frontmatter. If you leave Obsidian, you lose the views but keep 100% of your data

Example, a Projects base that auto-populates from frontmatter:
```yaml
filters:
  and:
    - file.hasTag("project")
    - 'file.ext == "md"'
properties:
  status:
    displayName: Status
views:
  - type: table
    name: Active Projects
```

Embed in any note with `![[Projects.base#Active Projects]]`.

Add Dataview only if you hit a wall needing inline queries inside a note body.

### Recommended: hotkey-passthrough (for terminal plugin users)

The [polyipseity/obsidian-terminal](https://github.com/polyipseity/obsidian-terminal) plugin embeds an xterm.js terminal inside Obsidian. When the terminal has focus, it captures **all** keyboard input, including Obsidian hotkeys like Cmd+O (Quick Open) and Cmd+P (Command Palette). The plugin offers no passthrough config. The only built-in escape is `Ctrl+Shift+`` ` `` to unfocus the terminal first.

**Fix:** A tiny custom plugin that registers a `keydown` listener in the DOM capture phase (fires before xterm.js's bubble-phase handler). When it detects a passthrough hotkey while an xterm element has focus, it stops the event and routes it to Obsidian's command system.

**Known gotcha:** The original v1.0 code had `!e.shiftKey` hardcoded, which silently blocked any Shift combos (e.g. Cmd+Shift+O for OmniSearch). Each hotkey entry must explicitly declare its `shift` value so the matcher compares rather than assumes.

**Installation, create two files in `.obsidian/plugins/hotkey-passthrough/`:**

`manifest.json`:
```json
{
  "id": "hotkey-passthrough",
  "name": "Hotkey Passthrough",
  "version": "1.1.0",
  "minAppVersion": "1.0.0",
  "description": "Ensures Obsidian hotkeys (Cmd+O, Cmd+Shift+O, Cmd+P) work when an embedded terminal has focus.",
  "author": "your-name",
  "isDesktopOnly": true
}
```

`main.js`:
```js
"use strict";

// Each entry: key (lowercase), meta, shift, ctrl, command ID to fire.
// Shift+key: e.key is uppercase in browsers, so normalize with toLowerCase().
const PASSTHROUGH_HOTKEYS = [
  { key: "o", meta: true,  shift: false, ctrl: false, command: "switcher:open" },
  { key: "o", meta: true,  shift: true,  ctrl: false, command: "omnisearch:show-modal" },
  { key: "p", meta: true,  shift: false, ctrl: false, command: "command-palette:open" },
];

class HotkeyPassthroughPlugin extends require("obsidian").Plugin {
  onload() {
    this._handler = (e) => {
      const active = document.activeElement;
      // Only intercept when focus is inside an xterm terminal
      if (!active || !active.closest(".xterm")) return;

      for (const hotkey of PASSTHROUGH_HOTKEYS) {
        if (
          e.key.toLowerCase() === hotkey.key &&
          e.metaKey  === hotkey.meta  &&
          e.shiftKey === hotkey.shift &&
          e.ctrlKey  === hotkey.ctrl  &&
          !e.altKey
        ) {
          e.preventDefault();
          e.stopPropagation();
          e.stopImmediatePropagation();
          this.app.commands.executeCommandById(hotkey.command);
          return;
        }
      }
    };
    // Capture phase fires before xterm.js's bubble-phase handler
    document.addEventListener("keydown", this._handler, true);
  }

  onunload() {
    if (this._handler) {
      document.removeEventListener("keydown", this._handler, true);
    }
  }
}

module.exports = HotkeyPassthroughPlugin;
```

Then add `"hotkey-passthrough"` to `.obsidian/community-plugins.json` and load via Settings → Community Plugins or:
```js
// Obsidian dev console
await app.plugins.loadManifests();
await app.plugins.enablePlugin('hotkey-passthrough');
```

**Adding more hotkeys:** Append entries to `PASSTHROUGH_HOTKEYS` with explicit `shift`/`ctrl` booleans. Find command IDs: `app.commands.listCommands()` in the dev console. Custom hotkey bindings (not defaults) live in `app.hotkeyManager.customKeys`.

### Recommended: obsidian-git

Install [obsidian-git](https://github.com/Vinzent03/obsidian-git) in every vault. This gives you automatic backup (commit + push on interval), pull on vault open, and a source control view inside Obsidian. Without it, git operations only happen from the terminal or Claude Code sessions.

Copy the plugin from an existing vault or install from Obsidian's community plugin browser. Make sure it's listed in `.obsidian/community-plugins.json`:
```json
["terminal", "obsidian-git"]
```

> **Note:** `.obsidian/plugins/` is in `.gitignore` (plugin binaries shouldn't be committed). Each machine installs its own copy. The `community-plugins.json` list *is* committed so Obsidian knows to load them.

### Plugins worth adding (when needed)

| Plugin | When to add | Why wait |
|--------|-------------|----------|
| **Templater** | When built-in templates feel limiting (need JS logic, prompts, auto-fill) | Built-in `Templates/` works fine for simple cases |
| **Tasks** | When action items get lost across files | Bases views can query tasks too — try that first |
| **Calendar** | When you have daily notes most days and want visual nav | Command palette creates daily notes fine for part-time use |
| **Pane Relief** ([pjeby/pane-relief](https://github.com/pjeby/pane-relief)) | When tab navigation feels clunky — per-tab back/forward history, hotkeys to move tabs left/right and to new windows, tab number jumping | Default tab behavior is fine for light use; install when you have many open tabs and keep getting lost |

### Tab management note

Multi-select tabs (select several tabs, then drag or close all at once) is **not a native Obsidian feature** as of 1.12. The only dedicated community plugin (`tab-multi-select`) was unreviewed and had 1 star as of May 2026, skip it. Pane Relief is the well-respected alternative for serious tab management.

### Plugins to skip

- **Dataview**, Bases covers it (see above)
- **tab-multi-select**, unreviewed and untested, use Pane Relief instead for tab management
- Anything that duplicates what Claude Code + scripts can do
- Anything that embeds proprietary syntax deep inside your content files

## Symlinked External Repos

A common pattern: a private Obsidian vault that wraps a shared repo, adding your own notes, people files, planning, and observations on top of content that lives in another repository you don't solely control.

### Setup

```bash
cd /path/to/vault
ln -s /path/to/shared-repo repo-name
echo "repo-name" >> .gitignore
```

### Rules
- Add the symlink name to `.gitignore`, the symlink target is its own repo with its own git history
- **Never modify files through the symlink**, edit the source repo directly. Obsidian doesn't distinguish symlinked files, so it's easy to accidentally edit shared content from a private vault
- Obsidian indexes symlinked content, so it shows up in search, graph view, and autocomplete, this is the point
- Document the symlink in `CLAUDE.md` and `README.md` so future sessions know the boundary

### Two-repo table in CLAUDE.md

When wrapping another repo, add a table to `CLAUDE.md` that makes the boundary explicit:

```markdown
## Two Repos — Keep Them Distinct

| | This vault | Source repo |
|---|---|---|
| **Repo** | private vault repo | shared/team repo |
| **Path** | /path/to/vault | /path/to/source-repo |
| **Contains** | Your notes, people, transcripts, planning | The actual project content |
| **Git** | Commit/push freely | Coordinate with collaborators |
```

### Multi-repo observatory variant

> Inspired by a course participant's fund-automation system, a production system where 3 people run a VC fund with 5 AI agent personas, 12+ scheduled tasks, and a voice-to-knowledge pipeline. The repo (`a private repo`) is no longer public.

When monitoring many repos at once (e.g., advising someone with multiple projects), group them under a single parent directory and use one symlink:

```bash
# Clone all repos into one parent
mkdir -p ~/Source/project-name
cd ~/Source/project-name
git clone https://github.com/org/repo-a.git
git clone https://github.com/org/repo-b.git
git clone https://github.com/org/repo-c.git

# One symlink exposes the whole tree to Obsidian
cd /path/to/vault
ln -s ~/Source/project-name sources/repos
echo "sources/repos" >> .gitignore
```

Adding a new repo is just `git clone` into the parent dir. No new symlink, no gitignore update, no vault config change. Obsidian auto-indexes it.

**Push-disable safety:** When monitoring someone else's repos (advising, reviewing, observing), disable push as a default to prevent accidental modifications:

```bash
for repo in ~/Source/project-name/*/; do
  git remote set-url --push origin DISABLE
done
```

Re-enable per-repo when you're ready to contribute: `git -C /path/to/repo remote set-url --push origin <url>`.

### No-duplication rule

The vault adds YOUR layer, notes, observations, people context, action items, teaching notes, decisions, on top of what's in the source repo. Never copy source content into the vault. Reference it via the symlink or wikilink to `repo-name/` paths.

If you need to annotate a specific file from the source repo, create a note in your vault that links to it:
```markdown
## Notes on [[repo-name/path/to/file|File Name]]
My observations about this file...
```

### Project notes for external repos

For each external repo you're tracking, create a committed `.md` file in `Projects/`. These notes persist in your vault's git even as the external repos change. The key sections:

- Frontmatter with `repo:` URL field
- Overview and architecture (stack, schedule, dependencies)
- Link to symlinked code: `[[sources/repos/repo-name]]`
- **Observations** section, dated notes that accumulate over time

For multi-repo projects, also create a system-level overview that describes how the repos fit together, architecture layers, data flow, dependencies, roadmap priorities.

### Staying current with the source repo

**Foundation: digest script** (tier 1, repeatable). Create a script that pulls and summarizes recent changes across all repos:

```bash
# scripts/source-repo-digest.sh
#!/bin/bash
set -euo pipefail
DAYS="${1:-7}"
REPOS=(
  "/path/to/source-repo-a"
  "/path/to/source-repo-b"
)
for REPO in "${REPOS[@]}"; do
  NAME=$(basename "$REPO")
  echo "=== $NAME ==="
  cd "$REPO"
  git pull --ff-only 2>&1 || echo "(pull skipped)"
  echo "--- Commits in last $DAYS days ---"
  git log --oneline --since="$DAYS days ago" --no-merges
  echo ""
done
```

If you track many repos, list them in a `repos.conf` and have the digest and catch-up scripts read from it, so adding a repo is a one-line change instead of a script edit.

**Recommended: `/catch-up` command** (tier 3, skill). Wrap the digest script in a Claude Code command that also analyzes what changed, syncs findings into the vault, and creates/updates the daily note.

The `/catch-up` command is more useful than the raw script because it reads actual diffs (not just commit messages), cross-references changes against project context, and writes structured summaries. Use the script for a quick pull, the command for full session context.

**Optional: `/catch-up-insights`** for periodic deeper analysis, narrative of what changed and why, security concerns from recent commits, patterns and learnings.

Add to `CLAUDE.md` under a **Session Start** section:
```markdown
## Session Start
1. Run `/catch-up` to pull all repos, analyze changes, and get current context
2. Or for a quick start: run `./scripts/source-repo-digest.sh` to just pull and show recent changes
```

This way you always know what changed in the source repos without having to remember to check, and the vault stays a clean overlay, never a fork.

## Per-Vault Theming from Claude Code

Each vault has its own `.obsidian/` directory, so every vault gets a completely independent look. Give each vault a distinct identity so you instantly know which context you're in.

### The script (tier 1 — repeatable)

```bash
scripts/theme-vault.sh /path/to/vault <preset> [accent-hex]
```

Available presets:

| Preset | Vibe |
|--------|------|
| `earthy` | Warm dark, sandy accent, muted teal links |
| `ocean` | Cool dark, deep blue accent, soft cyan links |
| `forest` | Dark green undertones, sage accent, warm highlights |
| `slate` | Neutral gray, minimal color, clean and quiet |

The script handles everything: downloads the Catppuccin base theme, writes a preset-specific CSS snippet, and configures `appearance.json` with fonts and accent colors. Reload Obsidian to see changes.

### Accent is the identity, so it has to be unique

The accent color is what your eye actually reads in Mission Control, `Cmd+Tab`, and a second monitor. It is also the one thing four presets cannot give nine vaults. Pass `accent-hex` as a third argument to keep the preset's backgrounds but claim your own accent.

Two notes before re-theming an existing vault. The script replaces `appearance.json` wholesale, so it drops a copy at `appearance.json.bak` first — if a vault has a hand-built look (its own snippet, no Catppuccin, `showRibbon`), leave it alone and just add an `accentColor` key rather than running the script over it. And the window title already carries the vault name natively, so no plugin is needed for the `Cmd+Tab` case; color is the part Obsidian won't do for you.

## Window Identity Across Obsidian and Your Editor

Same problem, two apps: many windows open, no way to tell which is which from across the room. Obsidian solves it with `accentColor`; VS Code and its forks (Antigravity IDE, Cursor, Windsurf) solve it with `workbench.colorCustomizations` in a workspace's `.vscode/settings.json`. The Peacock extension is a UI over exactly those keys — worth skipping, since writing them directly needs no extension and no update to break.

`scripts/workspace-identity.py` covers both:

```bash
scripts/workspace-identity.py audit ~/Source/obsidian ~/Source   # check
scripts/workspace-identity.py set ~/Source/my-repo '#4c8f7d'     # claim a color
scripts/workspace-identity.py label ~/vaults/course 'Course - My Private Vault'
scripts/workspace-identity.py --self-check                       # prove it works
```

`audit` lists every project's color and exits nonzero if two share one:

```
the client vault          obsidian  #6b8fad
Music                  obsidian  #6b8fad   COLLIDES with the client vault (obsidian)
```

Run it after adding a vault or a repo. A collision is invisible from inside either project — each one looks correctly configured on its own, and the confusion only shows up when both windows are open, which is the exact moment the color was supposed to help. See `patterns/identity-is-a-property-of-the-set.md`.

Two deliberate asymmetries in what `audit` treats as a finding. A directory with `.obsidian/` but no accent **is** a finding, because Obsidian creates that folder for every vault, so its presence means "this is a vault that should be recognizable." A repo with no `.vscode/settings.json` is **silent**, because it has never been opened in an editor and flagging every checkout on disk would bury the real findings.

`set` writes whichever surfaces a project has — `accentColor` for a vault, `workbench.colorCustomizations` for an editor workspace, both if it is both — merging into what is already there and picking black or white title text by luminance so the bar stays readable. A vault that enables a `vault-*` preset snippet is the one case `set` can't finish: the snippet hardcodes the accent it was generated with and would win, so `set` says so and points at `theme-vault.sh <vault> <preset> <hex>`.

### Say the name, not the folder name

Color tells you which window from across the room. It does not tell you *which of the two the course projects this is* — and folder names are no help when the private planning vault is `the course` and the repo shared with a co-instructor is `the course repo`. `label` puts a human name where you actually look:

```bash
scripts/workspace-identity.py label ~/vaults/course 'Course - My Private Vault'
scripts/workspace-identity.py label ~/src/course-repo 'Course - Shared Repo'
```

In a vault it writes a `vault-label` CSS snippet — a bold badge pinned to the bottom-left of the window in that vault's own accent — and enables it *alongside* whatever snippets were already on rather than replacing them. It is pinned to the window rather than hung off the sidebar on purpose: the first version anchored to the left split, so collapsing the sidebar hid the label while every file on disk still looked correct. This is the Vault Name plugin's job done by the snippet mechanism the vault already has, so there is no plugin to break on an Obsidian update. In an editor workspace it sets a workspace-level `window.title`, which outranks the user-level one below.

`theme-vault.sh` carries the label snippet across a re-theme. Any *other* hand-enabled snippet has to come back from `appearance.json.bak` — the script stays curl-and-bash and does not parse JSON to read them.

### The `Cmd+Tab` half, in the editor

VS Code and its forks show the file name first by default, so every window looks alike in the switcher. One user setting fixes it for all workspaces at once — no extension:

```json
"window.title": "${rootName} - ${activeEditorShort}"
```

On macOS that lives in `~/Library/Application Support/<ProductName>/User/settings.json` (`Antigravity IDE`, `Cursor`, `Code`). Obsidian already puts the vault name in its window title, so this is the editor catching up, not a new idea.

Keep `.vscode/settings.json` out of shared repos with `.git/info/exclude` rather than `.gitignore` — it is per-clone and never committed, so a personal color preference does not land in a teammate's diff.

### How it works under the hood

- **Community themes** live in `.obsidian/themes/<ThemeName>/`, requires both `theme.css` and `manifest.json` (both files needed or Obsidian won't recognize the theme)
- **CSS snippets** live in `.obsidian/snippets/`, load after the theme (CSS cascade), so they override it
- **`appearance.json`** ties it together, `cssTheme`, `theme` (`"obsidian"` = dark, `"moonstone"` = light), fonts, `accentColor`, `enabledCssSnippets`

### Customizing beyond presets

Edit the generated snippet in `.obsidian/snippets/vault-<preset>.css` directly, or create additional snippets. Key CSS variables:

- `--background-primary/secondary`, main backgrounds
- `--text-normal/muted/faint`, text colors
- `--interactive-accent`, accent color (buttons, selections)
- `--link-color` / `--link-external-color`, internal vs external links
- `--h1-color` through `--h6-color`, heading colors
- `--font-text` / `--font-monospace` / `--font-interface`, font overrides in CSS

Fonts can also be set in `appearance.json` via `textFontFamily`, `monospaceFontFamily`, `interfaceFontFamily` (comma-separated, uses system-installed fonts).

### Adding new presets

Add a new case to `scripts/theme-vault.sh`. Each preset is just a set of color values passed to `write_snippet`. Keep it simple, pick a palette, fill in the 15 color slots.

## What This Setup Avoids

- Over-processing simple inputs
- Asking unnecessary clarifying questions
- Creating files that should be scripts
- Duplicating content across files
- Inline tags
- Plain text where wikilinks should be
- AI-sounding language in drafts
- Emojis unless asked
- Time estimates

## Overriding Default Claude Code Harness Behaviors

Claude Code ships with default behaviors baked into the harness system prompt that you can't see directly but that show up in every session. Some of them are useful, some are noise. They include things like a default offer to `/schedule` a recurring agent at the end of replies that have a "follow-up signal," default offers to summarize at the end of long sessions, default phrasings that come back even when you've asked for terseness elsewhere.

User instructions take precedence over the default system prompt. So the cleanest way to suppress a behavior you don't want is a one-paragraph rule in your global `~/.claude/CLAUDE.md`. Be specific about what you're suppressing. Vague instructions ("be concise") don't override defaults reliably, named instructions ("never end replies with X") do.

### Template

```markdown
## Never <do specific behavior>

The default Claude Code system prompt instructs the assistant to <describe the behavior>. Suppress this entirely. Do not <action 1>, do not <action 2>, do not <action 3>. If <user> wants <behavior>, they'll ask. <How to end replies cleanly instead>.
```

### Common candidates

- **`/schedule` offers**, the default prompt nudges Claude to end replies with a one-line offer to schedule a background agent when work has a follow-up signal (feature flag, soak window, recurring sweep). If you don't use scheduled agents, this is noise on every reply
- **End-of-session summaries**, if you've already asked for terseness, suppress the auto-summary too
- **Hedging phrases**, "let me know if I can help with anything else," "happy to dive deeper," etc. Specific phrases override the harness defaults better than abstract "be terse" rules

### Why specifics work

The harness defaults are themselves specific instructions ("end replies with X when Y"). Specific user rules ("never end replies with X") match the same pattern and cleanly override. Abstract rules ("be concise") set a tone but don't override individual behaviors.

If a behavior keeps coming back despite your instruction, your override is too vague, name the exact phrasing or action you want suppressed.

## Advanced Patterns: Automation Vaults

> Patterns in this section were derived from a course participant's fund-automation system, a production system where 3 people run a VC fund with the throughput of 10, powered by 5 AI agent personas, 12+ scheduled tasks, and a voice-to-knowledge pipeline. The repo (`a private repo`) is no longer public.

These patterns apply when an Obsidian vault wraps a system with automated pipelines, scheduled agents, or multiple data sources. Not every vault needs them, but when you're advising on or building complex automation, they're battle-tested.

### Deterministic code owns the database, LLMs own analysis

The most important architecture principle for mixed human/AI systems. Separate your automation into two layers:

- **Capture layer** (scripts, cron/launchd): Deterministic code that logs, archives, and syncs on fixed schedules. These own all database writes. The data store stays consistent even when the AI is slow, flaky, or wrong
- **Analysis layer** (Claude Code skills/commands): LLM-powered tasks that read the captured data, find patterns, generate briefings, and produce the human interface. If these fail, the worst outcome is a missed notification, not corrupted data

This maps to the priority hierarchy: capture = tier 1 (scripts), analysis = tier 3 (skills). Never let a tier 3 process write to the authoritative data store.

### Agent personas as Claude Code skills

Instead of building standalone agent apps, define agent personas as `SKILL.md` files. Each persona has:
- Defined responsibilities and decision frameworks
- Tool access declarations (which MCP servers, which data sources)
- Voice/personality guidance
- A "before starting" protocol that loads lessons from prior runs

The advantage: all agents share the same tool ecosystem. No separate deployments, no API wrappers. Adding a new agent is creating a new SKILL.md file.

**Skill-improver feedback loop:** Each agent maintains a Lessons Ledger (a markdown file recording what worked and what didn't). Before each run, the agent reads its ledger. After each run, a skill-improver protocol captures new lessons. This is an implementation of the LLM-wiki pattern (see `patterns/llm-wiki-maintenance.md`), establishing persistent agent memory via plain markdown, no database required.

### Local audio transcription for voice capture

Voice memos are high-value input for knowledge vaults. Capture a thought on your phone, have it transcribed locally, then route the text into the vault (daily note, project file, or a dedicated thoughts directory). Running transcription locally means no API costs, no data leaving your machine, and no rate limits.

**mlx-whisper (Apple Silicon)**, the best option for Mac. Runs Whisper models natively on the Neural Engine via Apple's MLX framework. Significantly faster than CPU-based alternatives and comparable to GPU inference.

```bash
pip install mlx-whisper
```

Minimal transcription script (tier 1, deterministic):

```python
import mlx_whisper
from pathlib import Path

# Available models (mlx-community HuggingFace repos):
# tiny, base, small, medium, large-v2, large-v3
# Start with large-v3 for accuracy, drop to small if speed matters more

def transcribe(audio_path: str, model_size: str = "large-v3") -> str:
    result = mlx_whisper.transcribe(
        audio_path,
        path_or_hf_repo=f"mlx-community/whisper-{model_size}-mlx",
        verbose=False,
    )
    return result["text"].strip()
```

**faster-whisper (cross-platform)**, works on Linux/Windows with CUDA GPUs, or CPU-only as fallback. Uses CTranslate2 for optimized inference. Heavier setup than mlx-whisper but runs anywhere.

```bash
pip install faster-whisper
```

```python
from faster_whisper import WhisperModel

model = WhisperModel("large-v3", compute_type="auto")
segments, info = model.transcribe("audio.m4a")
text = " ".join(s.text for s in segments).strip()
```

**whisper.cpp**, if you want a standalone binary with no Python. Compiles to native code, supports Apple Silicon acceleration via Core ML. Good for shell scripts and launchd jobs.

**Integration pattern for vaults:**

1. Watch a folder for new audio files (iCloud sync from a phone recording app, or a local drop folder)
2. Transcribe each new file locally
3. Write the transcript to the vault as a markdown note with YAML frontmatter (date, source, duration, tags)
4. Optionally: run Claude over the transcript to clean up, categorize, extract action items, and route content to the right vault locations
5. Schedule via launchd/cron (capture layer), let Claude handle analysis (analysis layer)

The participant's system system's memory pipeline (also no longer public) is a production example of this pattern: Just Press Record on iPhone → iCloud sync → mlx-whisper transcription → Claude enrichment → Obsidian vault with wikilinks and thesis tracking.

**Model selection:**

| Model | Size | Speed (Apple Silicon) | Accuracy | Use when |
|-------|------|----------------------|----------|----------|
| `tiny` | 75MB | ~30x realtime | Low | Testing, quick prototypes |
| `small` | 460MB | ~15x realtime | Good | High-volume batch processing |
| `medium` | 1.5GB | ~8x realtime | Very good | Daily use, most voice memos |
| `large-v3` | 3GB | ~4x realtime | Best | Default choice, handles accents and noise well |

First model download is slow (pulls from HuggingFace). After that, models are cached locally.
