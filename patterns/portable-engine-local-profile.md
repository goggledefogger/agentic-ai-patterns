---
type: pattern
date: "2026-08-02"
source: A personal skills repo (light-ops) / the personal agent
tags:
  - architecture
  - skills
  - configuration
  - portability
---

# Portable Skill Engine with Self-Scaffolding Local Profile

Author AI agent skills as 100% portable, distribution-ready engines, paired with an interactive or automated initialization step that discovers local infrastructure and generates a machine-local profile (`profile.yaml` or `*.profile.yaml`) for runtime execution.

## The Problem

When you build an AI agent skill that interacts with physical hardware, private services, or personal workflows (smart lights, local backup endpoints, printer queues, personal calendar signals), you run into a design split:

1. **Hardcoding local values:** If you write your own IP addresses, room names, and family members directly into the skill, the skill becomes private to your machine. You cannot share it to a public catalog or team repo without leaking private topology or breaking for everyone else.
2. **Abstracting into empty prompts:** If you make the skill 100% generic with no saved state, the AI agent has to ask you for your IP addresses, pinouts, and zone mappings every single session, or guess and fail.

## The Pattern

Split the skill into two clean layers:

1. **The Portable Engine:** The `SKILL.md` instructions, CLI tools, and reference guides in your skills repo stay completely generic. They accept standard command-line flags and know how to find a local configuration file in standard locations (current directory, `~/.config/<skill>/profile.yaml`, or environment variables).
2. **The Self-Scaffolding Profile Wizard:** The skill provides a zero-dependency setup command (e.g., `uv run init_profile.py` or `<tool> init`) that probes the local environment (mDNS, local subnet, Bluetooth, USB devices) and writes a personalized data file (`light-ops.profile.yaml`).

```
[Portable Skill Repo] (a personal skills repo)
  ├── SKILL.md (Generic orchestrator instructions)
  ├── scripts/engine.py (Config-aware CLI)
  └── scripts/init_profile.py (Self-scaffolding discovery wizard)
            │
            ▼ (Run once per machine or project)
[Local Machine / Vault / Project Root]
  └── light-ops.profile.yaml (Personalized IPs, zones, hardware pinouts, signal aliases)
```

### 1. Build the Discovery Step
Write a setup script that checks local network broadcasts (mDNS, SSDP, BLE) or prompts for initial endpoints, then outputs a human-readable YAML profile with sensible defaults.

### 2. Auto-Discover the Local Profile in the Engine
Make all CLI scripts check a standard hierarchy of paths:
- Explicit CLI argument (`--config /path/to/profile.yaml`)
- Environment variable (`SKILL_PROFILE_PATH`)
- Current working directory (`./light-ops.profile.yaml`)
- User config directory (`~/.config/light-ops/profile.yaml`)

If no profile is found when a command runs, print a one-line hint pointing to the init wizard rather than crashing with an unhandled exception.

### 3. Consume via Semantic Aliases
Once the profile is generated, agent instructions and human commands use meaningful high-level aliases instead of raw IP addresses:
```bash
# Before: Brittle, hardcoded or prompted every session
uv run scripts/wled_ctl.py --host 192.168.1.142 set --color "255,0,0"

# After: Portable engine reads local profile.yaml automatically
uv run skills/light-ops/scripts/wled_ctl.py signal meeting_on_air
```

## Why It Works

- **Publicly shareable:** The skill repo has zero secrets, zero private hostnames, and zero machine-specific paths. It can be pushed directly to a public or team repo
- **Zero-drift onboarding:** A new user (or you on a new machine) runs one init command to scan and scaffold their local setup in seconds
- **Deterministic runtime:** The agent reads structured YAML rather than trying to remember hardware details across conversational context windows

## When to Use

- Hardware and IoT automation skills (WLED, Zigbee, BLE, 3D printers, Home Assistant)
- Local developer tooling that binds to custom ports, local DB paths, or staging URLs
- Multi-agent systems (like the personal agent and the household agent) where an orchestrator dispatches tasks to machine-specific workers

## The Failure Mode: the profile reaches the plumbing, not the behaviors

Observed live 2026-08-27 (the dashboard, member-zero trial, the course repo, issue #174): a topology profile (`MEMORY_ROOTS`, PATH-style env naming a brain's extra memory stores) was adopted by every *infrastructure* consumer — file watchers, residue, backup, project listing all aggregated across stores — while the *behavioral* layer (the ritual instructions the spawned agent follows) was never told the variable existed. The agent, pointed at 1,954 notes across three stores, read only its empty cwd and reported "nothing to draw from."

The trap: adoption looks complete because the visible surfaces (dashboards, status lines) honor the profile. **Audit adoption per consumer, not per system** — every place that reads the topology must be listed, and an instruction file a child agent follows is a consumer exactly as much as a watcher is. The fix is the `seed-the-precondition-in-the-file-the-child-reads` move: name the profile variable in the instructions the child actually loads, and have the spawner state the topology outright when it is non-default.

## Source


Extracted during the creation of `light-ops` across `a personal skills repo` and the personal agent ambient lighting orchestration project.
