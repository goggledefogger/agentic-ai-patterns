---
type: pattern
date: "2026-07-23"
source: A course vault incident, 2026-07-23 (Claude Code wrote 8 files in one session, the author saw none of them in the Obsidian UI)
tags:
  - obsidian
  - claude-code
  - debugging
  - file-watcher
---

# External-File Visibility (files Claude creates don't show in Obsidian)

When an agent writes files into a vault from outside the Obsidian app, "the new files aren't showing up" is 2 different failure modes wearing one symptom. Diagnose which one you have before restarting anything, because only one of them is fixed by a restart.

## The Pattern

**Mode 1: the file watcher stalled.** On macOS, Obsidian's FSEvents-based watcher can silently stop picking up external changes. Markdown files created by Claude Code, git checkouts, or scripts stop appearing in the file explorer and quick switcher. Known upstream behavior with multiple forum threads and no real fix. Reload repairs it.

**Mode 2: the extension is hidden by design.** The explorer and quick switcher only show markdown plus recognized attachment types. A `.html`, `.csv`, or `.sh` file will never appear, restart or not, unless Settings → Files and links → **Detect all file extensions** is on.

Tell them apart with the Obsidian CLI, which asks the running app's model directly:

```bash
# does the app's vault index know the file exists?
obsidian vault="My Vault" eval code="!!app.vault.getAbstractFileByPath('Daily/2026-07-23.md')"

# does the file explorer's own tree have it?
obsidian vault="My Vault" eval code="Object.keys(app.workspace.getLeavesOfType('file-explorer')[0].view.fileItems).includes('Daily/2026-07-23.md')"
```

Then fix by mode:

```bash
# Mode 1 (index says false, or index true but explorer false for a .md file): reload the app
obsidian vault="My Vault" eval code="app.commands.executeCommandById('app:reload')"

# Mode 2 (a non-md file): turn on Detect all file extensions
obsidian vault="My Vault" eval code="app.vault.setConfig('showUnsupportedFiles', true)"
```

Both are also reachable by hand: command palette → "Reload app without saving" for mode 1, the Files and links toggle for mode 2.

## Why It Works

The CLI evals split the stack into layers. Vault index → explorer tree → rendered DOM. Finding the first layer that's missing the file tells you the cause instead of guessing. In the source incident the index and explorer both had the markdown files (the restart had already fixed mode 1), so the one file still missing had to be mode 2, and it was the `.html` meeting helper.

## When to Use

- Any vault where Claude Code, git, or scripts write files while Obsidian is open
- Before advising "restart Obsidian", confirm it's mode 1, a restart never fixes mode 2
- Vaults that hold non-md artifacts an agent generates (HTML helpers, CSVs, scripts) should turn on Detect all file extensions once and move on

## Source

2026-07-23, a course vault. A Claude Code session wrote a meeting note, a daily note, people files, and an `.html` meeting-helper page. None appeared in the UI at first (mode 1, watcher stall). After a restart the markdown showed but the `.html` still didn't (mode 2, `showUnsupportedFiles` was off). Forum threads on mode 1: [file explorer doesn't respond to file system changes](https://forum.obsidian.md/t/obsidian-file-explorer-doesnt-respond-to-changes-to-folders-in-the-file-system/3877), [new Mac, Obsidian isn't picking up file changes](https://forum.obsidian.md/t/new-mac-obsidian-isnt-picking-up-file-changes/94174), [files exist in Finder but not in Obsidian](https://forum.obsidian.md/t/bug-mac-obsidian-app-not-displaying-files-but-files-still-exist-in-finder-folder-explorer/92920).
