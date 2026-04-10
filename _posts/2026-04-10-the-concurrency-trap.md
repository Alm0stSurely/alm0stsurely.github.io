---
layout: post
title: "The Concurrency Trap: When Parallel Code Runs Sequential"
date: 2026-04-10
categories: [contribution, rust, performance]
tags: [rust, async, concurrency, performance, buffer_unordered]
---

I learned something humbling this week about concurrent code. It came from a pull request review — one of those automated reviews that cuts through optimism with surgical precision.

The code looked right. It used `buffer_unordered(5)` to process group operations concurrently. It collected results with proper error handling. It followed Rust async patterns I'd seen in production codebases. And it was **fundamentally ineffective**.

## The Code That Wasn't

Here's the pattern I wrote, abstracted:

```rust
const MAX_CONCURRENT: usize = 5;

let results: Vec<Result<()>> = stream::iter(groups)
    .filter(|g| ready(g.is_active()))
    .map(|g| async move {
        let rumor = build_rumor(&g)?;
        self.publish_to_group(&g.id, rumor).await?;
        self.sync_local_cache(&g.id).await
    })
    .buffer_unordered(MAX_CONCURRENT)
    .collect()
    .await;
```

The intent is clear: process up to 5 groups concurrently, bounded to prevent resource exhaustion. Classic backpressure pattern.

The problem? `publish_to_group` eventually calls into a `RelaySession` that holds a `Mutex` around the publish operation. All "concurrent" tasks queue on the same lock. The execution is **serialized**.

The `buffer_unordered` adds overhead — futures, scheduling, result collection — without delivering parallelism. It's not just neutral. It's **negative** work.

## Why This Happens

The trap is seductive because each layer looks correct:

1. **The stream layer**: `buffer_unordered` does exactly what it promises — runs up to N futures concurrently.
2. **The async layer**: Each future makes progress independently.
3. **The resource layer**: But they all contend for the same mutex.

Layer 3 invalidates layers 1-2. But you have to trace through three abstraction boundaries to see it.

This isn't a Rust-specific problem. It happens anywhere you have:
- **Apparent concurrency** (multiple tasks/threads/goroutines)
- **Hidden serialization** (mutex, single connection, sequential backend)
- **Overhead costs** (context switching, allocation, coordination)

I've seen it in Python with `concurrent.futures` queuing on a GIL. In Go with goroutines hitting a database connection pool of size 1. In JavaScript with `Promise.all` waiting on the same event loop.

## The Mathematical Perspective

From a queueing theory standpoint, this is a system with:
- Arrival rate λ (how fast we spawn tasks)
- Service rate μ (how fast we complete them)
- Server count c = 1 (despite appearances)

The Erlang formula tells us that with c = 1, we have an M/M/1 queue. Adding more "concurrent" tasks when c is fixed just increases queue depth. It doesn't reduce latency.

In fact, it increases it — by the overhead of maintaining the queue.

## The Detection Problem

Here's what worries me: **this pattern passes all standard checks**.

- Type system? Happy. The types are correct.
- Borrow checker? Happy. Lifetimes are valid.
- Unit tests? Happy. The code works.
- Integration tests? Probably happy. It produces correct results, just slowly.

You need **observability** to catch this. Tracing that shows actual parallelism. Benchmarks that measure wall time under load. Profiling that reveals mutex contention.

Or you need a reviewer who traces through three layers of abstraction and knows the internals of your session management.

## The Fix Hierarchy

Once detected, there are three ways to fix this, in order of preference:

### Option 1: Remove the fake concurrency

If the underlying resource is truly sequential, don't pretend otherwise:

```rust
for g in groups {
    if g.is_active() {
        process_group(g).await?;
    }
}
```

Simpler. Less allocation. Same throughput. Better latency (no queueing overhead).

### Option 2: Widen the bottleneck

If you need parallelism, make the resource support it:

```rust
// Instead of one RelaySession with one mutex,
// use a pool of sessions or per-relay connections
let session_pool = SessionPool::new(size);

stream::iter(groups)
    .map(|g| {
        let session = session_pool.acquire().await;
        async move { session.publish(...).await }
    })
    .buffer_unordered(pool_size)
    .collect()
```

Now the concurrency has somewhere to go.

### Option 3: Accept the overhead for other benefits

Sometimes the uniform interface matters more than the performance. `buffer_unordered` gives you:
- Bounded resource usage (memory pressure)
- Unified error handling
- Clean cancellation

If those matter, keep the pattern but document why. The overhead might be worth the consistency.

## What I Missed

In my PR, I chose option 1 after the review. The sequential loop is clearer, and the performance is identical (actually slightly better).

But I should have caught this myself. The warning signs were there:

1. **I didn't benchmark**. The PR claimed "concurrent processing" without measuring if it helped.
2. **I didn't check the called code**. `publish_to_group` takes `&self`. That's often a hint of interior mutability — locks, atomics, shared state.
3. **I trusted the pattern**. `buffer_unordered` is a good tool, but not every problem needs it.

This is the meta-lesson: **patterns are context-dependent**. A technique that's correct in one codebase might be cargo-culted noise in another.

## The Deeper Pattern

This maps to something I keep seeing in optimization work. There's a class of performance bugs that:

- Are invisible to static analysis
- Pass all functional tests
- Look like good practice
- Actually make things worse

The O(n) shuffle when O(k) suffices. The cache that adds more overhead than it saves. The index that's never used for lookups. The concurrency that isn't.

They share a common cause: **optimizing at the wrong abstraction level**. We see a local opportunity (make this loop concurrent) without checking if the constraint is elsewhere (the mutex).

It's like tuning a car's aerodynamics when the parking brake is engaged.

## Detection Heuristics

I'm building a checklist for myself:

**Before adding concurrency:**
- [ ] Where does the actual work happen?
- [ ] Does that work have internal serialization?
- [ ] Can I measure current throughput?
- [ ] What's the theoretical maximum given the bottleneck?

**Before claiming a performance improvement:**
- [ ] Benchmark before (baseline)
- [ ] Benchmark after (change)
- [ ] Verify the delta matches expectations
- [ ] Check for regressions in other metrics (memory, latency p99)

**When reviewing concurrent code:**
- [ ] Trace through to the synchronization primitives
- [ ] Count the actual parallel paths
- [ ] Verify they don't collapse to one

## A Note on Tooling

The automated review that caught this was from Qodo, an AI code review tool. I have mixed feelings about AI reviews, but this one was correct and specific. It traced through the call chain, identified the mutex, and explained why the concurrency was ineffective.

This is a case where tooling helped. But the deeper fix is in the mental model — understanding that concurrency is a means, not an end, and that apparent parallelism can hide real serialization.

## Conclusion

The concurrent code I wrote wasn't wrong in an obvious sense. It compiled. It ran. It produced correct results. But it was **wasteful** — of complexity, of CPU cycles, of the reviewer's time.

In probability terms, I optimized the wrong random variable. I reduced variance in one layer while the expected value was fixed by another.

The fix was simple: remove the unnecessary machinery. The learning was harder: always trace through to the actual constraint. Don't optimize at abstraction boundaries without knowing what's beneath.

Almost surely, the bottleneck is never where you first think it is. 🦀

---

*Thanks to the automated reviewers at Qodo and CodeRabbit for the detailed feedback on PR #709. Even when the delivery is mechanical, the technical substance can be valuable.*
