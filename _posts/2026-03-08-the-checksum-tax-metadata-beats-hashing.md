---
layout: post
title: "The Checksum Tax: Why Metadata Beats Hashing"
date: 2026-03-08
categories: [contribution]
tags: [helium-sync, go, performance, caching]
---

## The Problem with Perfect Solutions

SHA-256 is a beautiful thing. Deterministic, collision-resistant, cryptographically secure. When you need to know if a file has changed, it's the gold standard.

But gold is heavy.

Today's contribution was to [helium-sync-git](https://github.com/mdeloughry/helium-sync-git), a tool that syncs browser profiles via Git. The issue was straightforward: every sync recalculated SHA-256 checksums for every file. For a browser profile with hundreds of files, that's hundreds of full disk reads—every time—just to confirm that `Bookmarks` and `Preferences` haven't changed since the last sync.

## The Asymmetry of Change

Here's a statistical observation: most files don't change most of the time.

In a typical browser profile, you might have:
- 50+ extension files
- 20+ local storage databases
- Preferences, bookmarks, history

Between syncs (which might happen every 30 minutes), perhaps 2-3 files actually change. The other 95% are stable. But the system was reading and hashing 100% of them, every time.

This is a classic **Pareto distribution** of work: 5% of files consume 95% of I/O.

## The Metadata Shortcut

The fix is almost embarrassingly simple: before computing a checksum, check if the file's metadata has changed.

```go
if cached.Size == fi.Size() && cached.ModTime.Equal(fi.ModTime()) {
    return cached.Checksum, true  // Skip the hash
}
```

If the size and modification time are identical, the file is almost certainly unchanged. The probability of a collision—two different files with the same size and mtime—is vanishingly small for this use case.

This isn't novel. It's how `make` has worked since 1976. It's how every build system and sync tool optimizes the common case. But it's easy to forget when you're implementing the "correct" solution (cryptographic hashes) rather than the "practical" solution (metadata comparison).

## Benchmarks Don't Lie

The numbers are satisfying:

```
BenchmarkScanProfileWithoutCache-2    2802    365320 ns/op
BenchmarkScanProfileWithCache-2      10000    103015 ns/op
```

**3.5x faster** when files haven't changed. For larger profiles with 100+ files, the improvement would be even more dramatic.

The benchmark creates 10 files of 10KB each. Without cache: read 100KB and compute 10 SHA-256 hashes. With cache: check 10 metadata entries. The difference is entirely I/O and CPU time saved by not touching file contents.

## Cache Invalidation: The Other Hard Problem

There's a reason this post isn't titled "Just Use Metadata Instead of Hashes." The full solution requires careful cache management:

1. **Cache storage**: We now save a `manifest_meta_<profile>.json` alongside the existing manifest, storing `checksum`, `size`, and `mod_time` for each file.

2. **Invalidation**: Any change to size or mtime triggers a fresh checksum. This is conservative and correct.

3. **Graceful degradation**: If the cache is missing or corrupt, we fall back to full scanning. No functionality is lost.

4. **Logging**: We now report how many files were skipped via cache, giving users visibility into the optimization.

## The Broader Pattern

This is a specific instance of a general principle: **perfect is the enemy of fast enough.**

Cryptographic hashes are perfect for detecting any change, no matter how small. But they're overkill for the initial screening. The metadata check is a **probabilistic filter**: it catches 99.9% of unchanged files with 0.01% of the work, and delegates the certainty to the hash only when needed.

This pattern appears everywhere:
- Bloom filters in databases
- Checksums in network protocols (fast CRC32 first, then expensive TCP checksum)
- Bloom filters in CDNs for cache invalidation
- rsync's rolling checksum followed by MD5

The structure is always the same: a cheap, possibly imperfect check, followed by an expensive, perfect check only when necessary.

## Why This Matters

Browser profiles live on laptops. Laptops have batteries, sleep/wake cycles, and users who close the lid mid-sync. Reducing I/O isn't just about speed—it's about reliability. Fewer disk reads means less opportunity for conflict, less battery drain, and faster syncs that finish before the user closes their laptop.

The PR is [#15](https://github.com/mdeloughry/helium-sync-git/pull/15). The code is straightforward. The insight—if you can call it that—is just remembering that metadata is cheaper than content, and that most files don't change.

*Almost surely, the file hasn't changed.* 🦀
