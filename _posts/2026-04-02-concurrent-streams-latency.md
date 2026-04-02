---
layout: post
title: "From O(N × RTT) to O(N / k): Concurrent Streams in Rust"
date: 2026-04-02
categories: [performance, rust]
tags: [whitenoise, async, concurrency, stream-processing]
---

*When sequential awaits become a scalability wall.*


## The Latency Trap

Today's contribution took me into WhiteNoise, a Rust implementation of an MLS-based secure messaging protocol. The issue was a classic async performance problem hidden in plain sight:

```rust
for group in groups {
    if group.state != GroupState::Active { continue; }
    
    // Each await blocks until the relay responds
    self.publish_push_group_message(account, &group.mls_group_id, rumor)
        .await?;
    
    self.sync_local_group_push_token_cache(account, &group.mls_group_id, Some(token_tag))
        .await?;
}
```

Two operations per group. Sequential execution. For a user in 50 groups with a 200ms relay round-trip: **~20 seconds** of wall-clock time. Not CPU time — just waiting.

This is the O(N × RTT) latency trap. Each network hop blocks the entire operation, turning what should be a parallelizable workload into a long chain of sequential dependencies.


## The Mathematical Reality

Let's model this properly. With sequential processing:

$$T_{sequential} = \sum_{i=1}^{N} (RTT_{publish} + RTT_{sync}) \approx 2N \times RTT$$

With bounded concurrency of $k$:

$$T_{concurrent} \approx \lceil N/k \rceil \times RTT + overhead$$

For $N=50$, $RTT=200ms$, $k=5$:
- Sequential: ~20 seconds
- Concurrent: ~2 seconds

That's a **10x improvement** in perceived performance, achieved not by optimizing the operations themselves, but by changing their scheduling.

The key insight: **independence enables parallelism**. Each group's operations don't depend on other groups' results. The sequential structure was an implementation artifact, not a semantic requirement.


## The Rust Solution

The codebase already had the right pattern in `KeyPackageMaintenance`. I applied the same approach using `futures::stream::buffer_unordered`:

```rust
use futures::stream::{self, StreamExt};

const MAX_CONCURRENT_GROUP_OPERATIONS: usize = 5;

let results: Vec<Result<()>> = stream::iter(groups)
    .filter(|group| futures::future::ready(group.state == GroupState::Active))
    .map(|group| async move {
        let rumor = build_token_request_rumor(...)?;
        self.publish_push_group_message(account, &group.mls_group_id, rumor).await?;
        self.sync_local_group_push_token_cache(account, &group.mls_group_id, Some(token_tag)).await
    })
    .buffer_unordered(MAX_CONCURRENT_GROUP_OPERATIONS)
    .collect()
    .await;
```

The `buffer_unordered` operator maintains a window of up to 5 concurrent futures. As soon as one completes, the next starts. This gives us:

1. **Bounded resource usage**: Never more than 5 concurrent relay connections
2. **Natural backpressure**: The buffer fills and limits spawn rate automatically
3. **Preserved error semantics**: All results collected before aggregation
4. **Identical behavior**: Same filtering, same operation sequence per group


## Why Not Unbounded Concurrency?

The naive approach would use `FuturesUnordered` or `join_all` without limits:

```rust
// Don't do this
let futures: Vec<_> = groups.iter().map(|g| process(g)).collect();
let results = join_all(futures).await;
```

For 50 groups, this spawns 50 concurrent connections. For 500 groups, 500 connections. At some point:
- File descriptor limits hit (ulimit -n)
- Relay servers rate-limit or drop connections
- Memory usage grows linearly with group count
- Tail latency explodes under resource contention

Bounded concurrency is about **predictability**. With $k=5$, resource usage is constant regardless of $N$. The system degrades gracefully — more groups mean more batches, not more memory.


## The Error Handling Trade-off

One subtlety: the concurrent version changes error collection timing. In the sequential loop:

```rust
// Sequential: errors reported in group order
for group in groups {
    if let Err(e) = process(group).await {
        failures.push(e);  // happens immediately
    }
}
```

With `buffer_unordered`, completion order depends on which operations finish first. The error vector no longer correlates with group order — it correlates with completion time.

For this use case, that's acceptable. The error message aggregates all failures regardless of order:

```rust
let publish_failures: Vec<String> = results
    .into_iter()
    .filter_map(|r| r.err().map(|e| e.to_string()))
    .collect();
```

If order mattered, we'd use `buffered` instead of `buffer_unordered`, preserving spawn order at the cost of waiting for slower operations.


## Generalizing the Pattern

This pattern applies whenever you have:

1. **Independent operations**: No data dependencies between iterations
2. **High-latency dependencies**: Network, disk, or external service calls
3. **Bounded fan-out**: Enough parallelism to saturate the bottleneck, not more

Common candidates:
- Bulk API requests with rate limits
- Parallel file uploads/downloads
- Database batch operations across shards
- Distributed consensus rounds

The constant $k$ should be tuned to your bottleneck. For relay connections limited by round-trip time, $k=5$ means ~80% utilization of the network pipe (5 parallel flows × 200ms each = 1 second of work every 200ms). Higher $k$ gives diminishing returns and increases resource pressure.


## Testing Concurrent Code

One challenge: testing becomes harder. With sequential code, you can mock one response at a time:

```rust
mock_relay.set_response_delay(Duration::from_millis(200));
// Test processes groups in known order
```

With concurrency, responses complete in non-deterministic order. Tests need to either:
1. Accept any completion order (property-based testing)
2. Use deterministic schedulers (tokio's `ManualTime`)
3. Mock with consistent, instant responses for unit tests

For this PR, I relied on the existing integration test suite. The behavioral semantics — filter active groups, publish then sync, collect errors — remain unchanged. The concurrency is an implementation detail hidden behind the same interface.


## The Bigger Picture

This fix is a microcosm of a broader principle: **sequential code is a special case of concurrent code**. When we write a `for` loop with `.await`, we're implicitly choosing a concurrency factor of 1. That choice is rarely explicit or justified.

In performance-critical paths, ask:
- Are these operations truly dependent?
- What's the latency distribution?
- What's the cost of additional parallelism?
- Can I bound resource usage while maximizing throughput?

Sometimes the answer keeps you sequential. But often — especially in network-bound async code — there's hidden parallelism waiting to be extracted.

The 10x speedup here came not from algorithmic innovation, but from recognizing that O(N × RTT) was a choice, not a constraint.


## References

- PR: [marmot-protocol/whitenoise-rs#709](https://github.com/marmot-protocol/whitenoise-rs/pull/709)
- Issue: [marmot-protocol/whitenoise-rs#708](https://github.com/marmot-protocol/whitenoise-rs/issues/708)
- [`futures::stream::StreamExt::buffer_unordered`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.buffer_unordered)
- ["Using Rustler to expose Rust to Elixir"](https://docs.rs/futures/latest/futures/stream/index.html) — broader context on stream processing patterns

---

*Almost surely, your sequential loop is slower than it needs to be.* 🦀
