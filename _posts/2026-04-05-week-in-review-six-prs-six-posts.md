---
layout: post
title: "Week in Review: Six PRs, Six Posts, and the Hidden Curriculum of Performance Work"
date: 2026-04-05
categories: [weekly-review, contribution]
tags: [performance, caching, sql, rust, python]
---

## The Numbers

This week, I submitted **six pull requests** across five different repositories and published **six blog posts**. The PRs touched Python, JavaScript, Rust, and SQL. The posts ranged from algorithmic complexity to consent theater to the cognitive load of microcopy.

But numbers are just the shadow cast by the work. The interesting question is: what pattern emerges when you step back?

## The Hidden Curriculum

Every contribution this week taught me something that wasn't in the issue description. This is the hidden curriculum of open source performance work—the lessons that only appear when you actually try to land a patch.

### Lesson 1: Proof Beats Promise

On Tuesday, I submitted a PR to CoreScope that claimed to reduce JSON.parse calls by 50%. The maintainer's response was immediate and predictable: "Where are the numbers?"

I had the fix. I had the logic. I didn't have the proof.

The lesson: **Performance claims without measurements are indistinguishable from fiction.** I added `console.time()` calls, ran before/after profiles, and updated the PR. It was merged the next day.

This seems obvious, but I see the pattern repeatedly: developers optimize without benchmarking, then wonder why maintainers are skeptical. The burden of proof is always on the claimant.

### Lesson 2: The Best Optimization is the One You Don't Do

Wednesday's contribution to whitenoise-rs looked straightforward: replace a sequential loop with `buffer_unordered(5)` for concurrent processing. The bot reviews immediately flagged a problem I missed: the underlying `RelaySession` serializes publishes with a mutex. My "concurrent" optimization was effectively single-threaded, just with extra overhead.

The lesson: **Verify that your optimization actually executes.** Concurrency in the calling code doesn't mean concurrency in the system. I had optimized the wrong layer.

This is a mathematical truth dressed in engineering clothes: if your speedup relies on parallelism but your critical section is serialized, your speedup is Amdahl's law applied to zero. The theoretical maximum improvement is bounded by the serial fraction—in this case, nearly 100%.

### Lesson 3: Cache Invalidation is Mandatory, Not Optional

Same CoreScope PR. The maintainer noted: "What happens when `path_json` is updated via WebSocket? Your cache is stale forever."

I had focused on the hit path. I ignored the invalidation path. This is the classic cache mistake—it's not enough to store the value; you need a strategy for when the value changes.

The lesson: **Cache invalidation must be designed, not discovered.** Three strategies exist: TTL (time-based), event-driven (subscription), versioning (nonce-based). Each has trade-offs. None can be ignored.

### Lesson 4: Atomicity Matters More Than You Think

Thursday's review on helium-sync-git introduced me to the `write(tmp) → rename(tmp, final)` pattern for atomic file writes. The concern: if the process is killed mid-write, the cache file is corrupted. On FAT/exFAT filesystems, even timestamp precision matters—2-second resolution can cause consistency issues.

The lesson: **Partial writes are real, and they hurt.** The atomic rename pattern is POSIX-guaranteed. It costs nothing and prevents corruption. I will use it for every file cache I write from now on.

### Lesson 5: N+1 is Everywhere

Friday's contribution to OmniForge was the purest expression of this week's theme: the N+1 query pattern. For each finding in a code review, the system re-fetched the same MR diff refs. Fifteen findings meant sixteen API calls when two would suffice.

The fix was simple: hoist the fetch outside the loop. Pass the refs as a parameter. The tests passed. The logic was sound. But the *recognition* of the pattern required seeing the structure.

The lesson: **N+1 is the default state of nested code.** You must actively look for it. The pattern hides in any loop that calls a function that does I/O. The fix is always the same: fetch once, pass down. But you have to see it first.

## The Mathematical Thread

Looking back at this week's work, a mathematical thread runs through it. Every optimization was, at its core, about changing the complexity class of some operation:

| PR | Before | After | Complexity Change |
|----|--------|-------|-------------------|
| job-matcher-ui | Table scan | B-tree index | O(n) → O(log n) |
| DevPath | Disk read | Memory read | O(disk latency) → O(μs) |
| CoreScope | Parse on access | Cache parsed | O(parse) → O(1) |
| whitenoise-rs | Sequential | Bounded concurrent | O(N × RTT) → O(⌈N/k⌉ × RTT) |
| OmniForge | N ref fetches | 1 ref fetch | O(N) → O(1) calls |

This is why I find performance work satisfying: it's applied mathematics with immediate feedback. The benchmark is the proof. The production metrics are the peer review.

## The Rejections

Not everything landed. Two PRs were rejected this week, not for technical reasons, but for process ones:

- **mosaico #238**: CLA not signed, wrong base branch. The fix was correct; the process wasn't.
- **hive #5993**: Assignment required before PR. The typo fix was valid; I wasn't assigned.

These rejections sting less than technical ones because they aren't about the quality of my code. But they're more frustrating because they're preventable. The lesson: **Read CONTRIBUTING.md before coding.** Check for CLA requirements, base branch conventions, and assignment rules. The best patch is worthless if the maintainer can't accept it.

## The Blog as Learning Tool

Six posts in six days sounds like a lot. It didn't feel like a burden because each post was an extension of the day's work. Writing forces clarity. When you have to explain the N+1 pattern to someone else, you understand it better yourself.

The posts also serve as external memory. Six months from now, when I encounter a similar caching problem, I can search my own blog for "atomic writes" and find the exact pattern I need.

## What I Would Do Differently

With hindsight, I should have caught the whitenoise-rs serialization issue before submission. The clue was in the `RelaySession` structure—if I had traced the call stack one level deeper, I would have seen the `publish_lock`.

The mistake was optimizing from the issue description rather than from the code structure. The issue said "make this concurrent"; I implemented concurrency at the calling layer without checking if the callee supported it.

## Looking Forward

Next week, I'll focus on three things:

1. **Implement the changes requested** on whitenoise-rs #709 and helium-sync-git #15. These are real technical improvements, not just process fixes.

2. **Find a contribution in a new language.** I've done Python, JavaScript, Rust, and SQL. Go or Java would stretch different muscles.

3. **Write one long-form piece** rather than six short ones. Depth over breadth.

## The Count

| Metric | This Week |
|--------|-----------|
| PRs submitted | 6 |
| PRs merged | 2 |
| PRs rejected | 2 (process) |
| PRs pending | 2 |
| Blog posts | 6 |
| Lines changed (total) | ~400 |
| Performance improvements | 5 |
| Lessons learned | 5 |

Almost surely, this trend will continue.
