---
type: pattern
date: "2026-04-18"
source: A course participant's home-agent system (the sensor service)
tags:
  - code
  - architecture
  - queue
  - decoupling
---

# Filesystem as the Queue (No Broker Needed)

When a sensor process produces events that a processor consumes, the default instinct is to reach for Redis, SQS, or a message broker. For single-host setups that's almost always over-engineered. The sensor writes JSON files to a directory. The processor reads from it. The filesystem is the queue.

## The Pattern

```
~/events/
├── queue/        ← normal-priority events, processor picks up on next cycle
├── hot/          ← hot-priority events, processor picks up immediately
└── archive/
    ├── 2026-04-17/
    └── 2026-04-18/   ← date-partitioned archive of processed events
```

- **Sensor** writes one JSON file per event into `queue/` or `hot/` using the atomic-write pattern (see `atomic-state-writes.md`)
- **Processor** lists the directory, reads each file, processes it, and moves it to `archive/YYYY-MM-DD/` when done
- **Routing by priority** is folder-based: `hot/` gets scanned every cycle, `queue/` on a slower cadence
- **Archive by date** means cleanup is "delete archive directories older than N days."

## Why It Works

- **Zero operational overhead.** No broker to install, monitor, or upgrade. No network hop. No "did the broker restart and lose messages."
- **Observable with `ls`.** `ls queue/ | wc -l` is the queue depth. `ls archive/2026-04-17/ | wc -l` is yesterday's throughput
- **Crash-safe.** Atomic writes guarantee no partial events. If the processor crashes mid-process, the event file is still there for the next run
- **Decoupled by time and process.** Producer and consumer never need to be alive simultaneously
- **Debuggable by a human.** Open the JSON file in an editor. Compare two events with `diff`. Replay by moving from `archive/` back to `queue/`

## When to Use

- Single-host or small-fleet systems where throughput is well under 100 events/second
- When producer and consumer run on different cadences (sensor every 5 min, processor every 30 min)
- When you want a recovery story that doesn't involve broker replay APIs

## When NOT to Use

- Multi-host distributed processing (NFS + file locking gets ugly fast)
- Sub-millisecond latency requirements
- High throughput (>1000 events/sec sustained). Filesystem metadata ops become the bottleneck

## Source

`~/src/participant-system/writers/event_writer.py`, the canonical example. `EventWriter.write()` handles priority routing (hot vs queue), atomic tempfile+rename, and fail-soft cleanup. `ensure_archive_dir()` builds the date-partitioned archive folder for the processor to move events into.
