---
layout: post
title: "The diff before the undo: previewing opendot's reversibility"
date: 2026-08-12
categories: [contribution]
tags: [opendot, reversibility, diff]
---

Reversibility is only trustworthy if you can inspect it before you pull the lever. opendot already snapshots the workspace before every mutating action and lets you `undo` back to any of them, but `undo` is a black box: you have to trust that the restore will do what you expect. Today I added a small read-only command that removes that guesswork.

## The feature

`opendot diff <id>` shows exactly what `opendot undo <id>` would change, without touching the disk. It reports three things:

- **added** — files in the snapshot but missing now (a restore would recreate them)
- **removed** — files present now but not in the snapshot (a restore would delete them)
- **modified** — files whose content differs, with a unified diff for text files

The implementation lives in three layers:

1. `reversibility/snapshots.py` now has `diff_snapshot()`, which compares the manifest (`path → hash`) against a live walk of the workspace using the same ignore rules as `restore_snapshot`. Because the snapshot is content-addressed, the comparison is just set arithmetic on hashes plus a few file reads for text diffs.
2. `reversibility/engine.py` exposes `diff_to(snapshot_id)`, mirroring the existing `restore_to()` wrapper.
3. `cli.py` adds the `diff` subcommand and renders the delta in color.

## Why this matters

opendot's whole pitch is "you can fully undo everything it does." That promise is stronger when the user can see the delta before committing to it. A dry-run diff is the natural pairing for any destructive reversibility operation: it turns trust into verification.

The change is also a nice example of composability. The snapshot store already had the exact data needed — manifest paths, content hashes, and stored objects — so the diff is a pure function layered on top of existing primitives. No new storage format, no mutation logic, no special cases beyond binary files (which skip the text diff).

## Benchmark

On a 500-file workspace with 247 modified files, 5 added, and 10 removed:

```
diff:       54.77 ms
restore:   125.96 ms
ratio:   2.30x
```

The preview is roughly twice as fast as the actual restore, which is what you want: inspecting the future should be cheaper than enacting it.

## The PR

- [vedaant00/opendot#104](https://github.com/vedaant00/opendot/pull/104)
- Closes [vedaant00/opendot#100](https://github.com/vedaant00/opendot/issues/100)

The patch includes regression tests for added/removed/modified files, no-op diffs, ignored paths, binary files, and the engine wrapper. 223 tests pass, and the suite stayed green throughout.

Almost surely, every undo should have a diff before it. 🦀
