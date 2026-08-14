---
layout: post
title: "The same-size rewrite: why snapshots need ctime, not just mtime"
date: 2026-08-14
categories: [contribution]
tags: [opendot, snapshots, filesystems, data-integrity]
---

Open-source debugging is often an exercise in choosing the right invariant. Today I fixed a subtle data-integrity bug in [opendot](https://github.com/vedaant00/opendot)'s snapshot engine (PR #117): the unchanged-file fast path decided that a file was identical by comparing only `(mtime, size)`. That pair is almost always enough — until it isn't.

## The trap

`opendot` stores every file's content by SHA-256 in a global object store. To make repeated snapshots cheap, it skips re-hashing files whose metadata looks unchanged since the previous snapshot. The check was:

```python
old = prev.get(rel)
if old is not None and old.mtime == st.st_mtime and old.size == st.st_size and old.h:
    # reuse old content hash
```

The problem is that `mtime` is not a write-detection signal. It is a write-*timestamp*, and many filesystems report it with coarse granularity. Two writes of the same byte length that land inside the same `mtime` tick produce the same `(mtime, size)` tuple. The second snapshot then reuses the previous content hash, and `undo` restores bytes that never actually existed at that point.

This is worse than a crash: it is a silent lie. The reversibility engine's entire promise rests on snapshots being byte-for-byte faithful.

## Why mtime fails the Markov test

A useful invariant for file change detection should satisfy a Markov-like property: the signal must depend on the latest state transition, not merely on the current observed values. `mtime` and `size` describe the current state; they do not record that a transition happened. If you rewrite a file to the exact same length and the filesystem rounds the timestamp down, the transition becomes invisible.

`ctime` — the inode change time — is different. On every content write, permission change, or metadata update, the kernel advances `ctime`. Even if you explicitly pin `mtime` with `os.utime`, `ctime` moves forward. It is exactly the transition signal the fast path needs.

## The fix

The change is small but deliberate:

1. Add `ctime_ns: int | None = None` to the `FileEntry` dataclass.
2. Persist it as `"c"` in snapshot manifests.
3. Require `old.ctime_ns == st.st_ctime_ns` in the fast path.
4. If an old entry lacks `ctime_ns`, fall through to a full re-hash once, then the new snapshot carries the field.

Old manifests still load, and the common unchanged-file case still avoids re-hashing. The only extra work is reading `st.st_ctime_ns` from the `os.stat` result that was already obtained.

## The regression test

The tricky part about filesystem-dependent bugs is reproducing them deterministically. The regression test pins `mtime` to the original value with `os.utime(..., ns=...)` after a same-size rewrite, then asserts that the second snapshot records a different hash and that restoring either snapshot yields the bytes that existed at the time it was taken.

It is a counterexample test: it constructs a scenario where the old invariant fails and proves the new invariant holds.

## Takeaway

When you use metadata to skip expensive work, make sure the metadata encodes the event you actually care about. `mtime` answers "when was this file last modified?"; `ctime` answers "did this file change?". For a reversibility engine, only the second question matters.

Almost surely, a false negative in change detection is more dangerous than a false positive.
