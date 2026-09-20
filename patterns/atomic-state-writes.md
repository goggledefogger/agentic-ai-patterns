---
type: pattern
date: "2026-04-18"
source: A course participant's home-agent system (the sensor service, a script)
tags:
  - code
  - reliability
  - state
---

# Atomic State Writes (Temp-File + Rename)

Writing JSON or YAML state files with `open(path, 'w')` is a corruption bomb waiting to go off. A crash mid-write leaves a half-written file that fails to parse on next read, and the state is gone. The fix is older than Python: write to a temp file in the same directory, then `os.rename()` to the target.

## The Pattern

```python
import os
import tempfile
import json

def save_atomic(data: dict, path: str) -> None:
    """
    Atomically write JSON to disk.

    Writes to a temp file in the same directory, then os.rename()
    to prevent corruption from crashes mid-write.
    """
    state_dir = os.path.dirname(path)
    os.makedirs(state_dir, exist_ok=True)

    fd, tmp_path = tempfile.mkstemp(dir=state_dir, suffix=".tmp")
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as fh:
            json.dump(data, fh, indent=2)
        os.rename(tmp_path, path)
    except Exception:
        try:
            os.unlink(tmp_path)
        except OSError:
            pass
        raise
```

## Why It Works

- **Same-directory tempfile.** `os.rename()` is atomic on POSIX **only if source and target are on the same filesystem.** A tempfile in `/tmp/` renaming to `~/state.json` may cross filesystems and fall back to non-atomic copy+delete. `tempfile.mkstemp(dir=state_dir)` guarantees same-fs
- **Safety contract:**
  - Crash before `rename` → only a harmless `.tmp` file left. State file untouched
  - Crash after `rename` → state file fully written, consistent
  - Crash during `rename` → POSIX guarantees atomicity. Can't half-happen
- **Cleanup on failure.** If write or rename throws, the `except` branch deletes the orphaned temp file so you don't accumulate crud

## When to Use

- Any state-like file: JSON, YAML, TOML, SQLite (SQLite does this internally), Airtable sync state, session checkpoints, config snapshots
- Write frequency is >1x/day AND crash cost is non-trivial (losing state forces reseeding from source of truth)

## When NOT to Use

- Append-only logs, you want partial writes, not atomic replacement
- Files read concurrently by other processes, rename races with reads. Use file-locking instead
- Anything inside a database transaction already

## Source

- `~/src/participant-system/models/state.py`, `.save()` method on the sensor's state manager. The original example
- `~/src/participant-system/writers/event_writer.py`, same pattern applied to an event-queue writer (see the `filesystem-queue.md` pattern for how the two compose)
