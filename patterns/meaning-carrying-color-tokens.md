---
type: pattern
date: "2026-07-22"
source: A shared tooling stack — brand pass across HTML artifacts and terminal scripts in one PR
tags:
  - design
  - frontend
  - terminal
  - branding
---

# Meaning-Carrying Color Tokens: One Semantic Palette Across HTML and Terminal

A tool that generates both HTML artifacts and terminal output got a branding pass. The trap on such passes is decoration: colors chosen per-surface, per-mood, carrying nothing. The fix that held up: define the palette once as *semantic* tokens — each color is a category, not a vibe — and echo the same semantics on every surface the tool speaks through.

Concretely: components of a tech stack got kind-colors (data = blue, access = ochre, tiling = brand orange, viz = green, standard = violet, infra = gray) as CSS custom properties in one shared Python constant, injected into every HTML template. The terminal scripts then used ANSI approximations of the same semantics (ok = green, broken = red, check = yellow, brand accent = orange 256-color). A reader who learns the color system in one place has learned it everywhere.

## The Pattern

- **One token constant, injected everywhere.** A single `_BRAND_CSS` string (or equivalent) holding `--accent`, neutrals, and the semantic category colors, substituted into every generated template. No template carries its own hex values; a rebrand is a one-constant diff.
- **Color = category, always.** If two things share a color they must share a meaning. If a color appears once, ask whether it should exist.
- **Terminal is a brand surface too.** TTY-gated ANSI (`[ -t 1 ]` — piped output stays byte-clean, an asserted invariant), the same semantic mapping, and at most a small ascii masthead. Cleverness budget: ~3 lines.
- **Lean is part of the brand.** System-font fallbacks over font files in the repo, one CSS transition with a `prefers-reduced-motion` guard over animation libraries, zero new dependencies. A brand pass that adds a framework failed.
- **Testable:** assert the token string appears in every generated surface, and that piped script output contains zero escape bytes. Brand consistency becomes a unit check, not a review comment.

The eval angle is what makes this durable: templates and scripts drift under parallel edits, and "does every surface still carry the token set" is exactly the kind of claim that silently rots unless a check pins it.
