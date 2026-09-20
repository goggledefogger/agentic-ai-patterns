---
type: pattern
date: "2026-08-24"
source: The personal agent's backup-ops skill, offsite backup audit — an abandoned restic repository on OneDrive holding 72.9 GiB across 4,447 objects and zero snapshots, 2026-08-24
tags:
  - backups
  - verification
  - data-integrity
  - storage
  - anti-pattern
---

# A Store Can Hold Data and Restore Nothing

Auditing an offsite backup, the personal agent found a restic repository on OneDrive holding 72.9 GiB across 4,447 objects, data and index files, fully intact, listable, sizable. It had zero snapshots. restic could not restore one byte of it. The repository was an interrupted first run: restic uploads packs before it writes the snapshot record that references them, so a killed run strands everything it already moved. Every plausible quick check, rclone's size report, an object listing, the directory just being there, reported a healthy-looking repo. Only `restic snapshots`, the reference layer, told the truth: an empty list. 72.9 GiB of dead weight had been sitting on a drive with 40 GiB free, looking like a backup, for two days.

## The Pattern

Any system with a reference layer over a storage layer, backup snapshots over packs, an index over blobs, a database catalog over pages, git refs over objects, can hold arbitrarily much real data that nothing references. Every storage-layer check passes on that data anyway: it's present, it's the right size, it lists fine. None of that is evidence it's recoverable. Recoverability lives in the reference layer, and only there.

Verify in that order:

1. **Count references first.** Snapshot count, index-entry count, ref count, row count in the catalog, whatever the system's "here is what you can get back" list is. Zero references means zero recoverable data, no matter how many gigabytes sit underneath.
2. **Walk one reference down to bytes.** A nonzero reference count proves something is listed, not that it resolves. Restore or read one referenced item through the real path to close the loop.
3. **Treat unreferenced storage as waste, not backup.** It passes every presence check and serves nobody. When auditing for space or auditing for safety, it's the first thing to name.

A write is complete when its reference lands, not when its bytes land. That ordering, bytes first, reference last, is exactly why an interrupted run strands the data: the run dies in the gap between the two, and the gap is invisible from the storage layer.

## Why It Works

Storage-layer checks are the cheap, obvious ones, and they return big reassuring numbers. "The destination holds 72.9 GiB" sounds like more evidence than "the repo lists 2 snapshots." It's strictly less. Size and object count answer "did bytes move." Only the reference layer answers "can anything come back," and that's the question a backup, an index, or a catalog exists to answer in the first place.

## When to Use

- Auditing any backup destination, restic, Borg, Duplicity, a plain rsync tree with a manifest, don't stop at size or file count. Check the snapshot/manifest list, then restore one item.
- Reviewing a search index, a cache, or a pipeline built as an index-over-blobs shape. The index's entry count is the thing to trust, not the blob store's byte count.
- Investigating a "storage keeps growing but nothing changed" report. Unreferenced data accumulating quietly is the first suspect.
- Any interrupted-write investigation where the destination looks intact. Check what got referenced, not what got copied.

## Adjacent Patterns

- `opaque-write-needs-a-read-back.md`, a read-back proves content; this pattern says which layer to read it back through, the reference layer, not the storage layer underneath it.
- `no-delivery-without-arrival-accounting.md`, the ledger analog: a channel needs an arrival signal the same way a store needs a reference count, and both fail silently without one.
- `ack-outran-the-write.md`, the ordering analog. There the ack must land last because it deletes the source; here the reference must land last because its absence orphans the write. Same discipline, mirrored: order the irreversible-meaning step after the durable one, and treat it as the true commit point.

## Source

The personal agent's backup-ops skill, offsite backup audit, 2026-08-24. `restic snapshots` against a OneDrive-hosted restic repository returned an empty list against 4,447 objects and 72.9 GiB of intact, listable pack and index files. rclone's size report, an object listing, and the directory's presence all read as a healthy repo; only the snapshot list showed zero. Diagnosed as an interrupted first backup run: restic uploads packs before writing the snapshot record that references them, so a killed run strands everything already moved, unreachable by any restore path.
