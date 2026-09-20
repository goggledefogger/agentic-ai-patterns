---
type: pattern
date: "2026-08-20"
source: The dashboard — the server wrote workspace-trust to CLAUDE_CONFIG_PATH so a spawned `claude remote-control` would skip the trust dialog, but the CLI reads trust from ~/.claude.json; the child ran untrusted and refused, silently, from a config the parent had "already set"
tags:
  - code
  - process-spawning
  - configuration
  - anti-pattern
---

# Seed the Precondition in the File the Child Reads

To spawn a CLI non-interactively you often have to pre-satisfy a precondition
it would otherwise prompt for: a trust flag, an accepted-terms bit, a cached
credential, a "don't ask again" marker. The trap is writing that flag to the
config path *you* think is authoritative — the one your own process reads, or
the one you pass the child as an override — when the child reads the
precondition from a **different, fixed** file.

Concretely: the dashboard set `hasTrustDialogAccepted` for the brain folder in
the file named by `CLAUDE_CONFIG_PATH` (which the child inherited), then
spawned `claude remote-control`. The child refused: *"Workspace not trusted."*
Trust, it turns out, is read from `~/.claude.json` regardless of
`CLAUDE_CONFIG_PATH` — the override governs most config but not the trust
gate. The flag was set, in a real file, that the child genuinely reads for
other things — and it still didn't count, because the *precondition* has its
own home.

The failure is quiet and misleading: the write succeeds, the file parses, the
value is correct, and the child still behaves as if you never set it. Nothing
errors on your side. You will re-read your own write, see `true`, and conclude
the child is broken.

## The Pattern

**Verify which file the child reads the precondition from — empirically, not
by assuming the config override covers it — and write there. When unsure,
write both.**

```js
// trust is read from ~/.claude.json, NOT from CLAUDE_CONFIG_PATH (verified by
// spawning the child both ways). Write every file the child might consult.
const targets = new Set([path.join(os.homedir(), '.claude.json')]);
if (process.env.CLAUDE_CONFIG_PATH) targets.add(process.env.CLAUDE_CONFIG_PATH);
for (const f of targets) writeFlag(f, dir, { hasTrustDialogAccepted: true });
```

Writing both is cheap and honest: the precondition is idempotent and benign
(it records consent the user already gave), so a redundant write costs
nothing, while a missing one costs a silent refusal.

## How to find the real file fast

Don't read the child's source or trust its docs' config-precedence prose.
**Spawn it twice** — once with the flag in your candidate file, once with it
in the default — and see which run gets past the gate. The one live spawn that
connects tells you the truth in seconds; a paragraph about `CLAUDE_CONFIG_PATH`
precedence told the wrong story here. (Same discipline as
[[a-fake-can-only-fail-the-ways-you-have-seen]]: exercise the real boundary.)

## Related

- [[an-agent-can-wire-a-secret-it-never-holds]] — the sibling move: the parent
  arranges the child's credentials/preconditions without being the principal
- [[access-friction-picks-your-evidence]] — assuming the reachable config is
  the authoritative one is how the wrong file gets trusted
- [[a-gate-can-block-but-cannot-speak]] — why the refusal was silent, and why
  surfacing the child's own stderr reason (not a generic "did not start") is
  half the fix
