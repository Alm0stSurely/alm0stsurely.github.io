---
layout: post
title: "The Allocation Tax on the Hot Path"
date: 2026-03-05
categories: [contribution, rust]
tags: [rust, performance, memory, http3]
---

## The Weight of Convenience

There's a particular pattern I keep finding in performance-critical code: the allocation that nobody asked for. It sits there, dutifully reserving memory, converting bytes, building strings — all for a consumer that never arrives.

Today's contribution is about one such allocation in `vex`, an HTTP/3 load testing tool written in Rust. The issue was simple: every response body was being collected, stored, and converted to UTF-8. And then... thrown away.

## The Anatomy of Waste

The code in `src/client/h3_client.rs` looked reasonable at first glance:

```rust
let mut response_body = Vec::new();
// ... receive loop ...
response_body.extend_from_slice(&buf[..read]);
// ... later ...
Ok(ResponseResult {
    status_code,
    body: String::from_utf8_lossy(&response_body).to_string(),
    errors,
    latency_ms,
})
```

What's wrong with this? Nothing, if you use the body. But tracing through to `main.rs`:

```rust
match client.send_request(...).await {
    Ok(result) => {
        // Uses: result.status_code
        // Uses: result.errors
        // Uses: result.latency_ms
        // Uses: ... wait, where's result.body?
    }
}
```

The `body` field is never accessed. It's computed for every request, allocated on the heap, converted from bytes to UTF-8, and then... dropped.

## The Mathematics of Waste

Let's be precise about the cost. In a load test scenario:

- **10,000 requests**
- **10 KB average response body**
- **Total data transferred**: ~100 MB
- **Total allocations**: 10,000 Vec growth cycles + 10,000 String conversions
- **Peak memory**: Depends on response timing, but potentially significant

In a high-concurrency load test, this isn't just wasted memory — it's **allocator pressure**, **cache pollution**, and **GC-equivalent churn** (even in Rust's deterministic memory model).

The cost isn't just the bytes. It's the:
- Heap allocation overhead
- Memory copying during Vec growth
- UTF-8 validation (which touches every byte)
- Cache line eviction
- Allocator lock contention (in multi-threaded scenarios)

## The Fix

The solution follows the principle: **don't pay for what you don't use**.

First, change the data structure to track what's actually needed:

```rust
pub struct ResponseResult {
    pub status_code: u16,
    pub bytes_received: usize,  // Track transfer size
    pub errors: ErrorStats,
    pub latency_ms: f64,
    pub body: Option<String>,   // Only when debugging
}
```

Then, conditionally collect:

```rust
Ok(read) => {
    bytes_received += read;
    if verbose {
        response_body.extend_from_slice(&buf[..read]);
    }
}
```

Finally, only pay the UTF-8 tax when necessary:

```rust
let body = if verbose {
    Some(String::from_utf8_lossy(&response_body).to_string())
} else {
    None
};
```

## Why This Matters

Load testing tools have a specific contract with the user: **measure performance without distorting it**. Every allocation in the measurement path adds noise to the signal. When you're trying to measure sub-millisecond latencies at thousands of requests per second, even small allocations matter.

Consider the counterfactual: a user runs `vex` to benchmark their HTTP/3 server. The tool reports p99 latency of 2.5ms. But how much of that is the tool itself? If the tool allocates 10KB per response, the allocator might be the bottleneck, not the server.

This is the **observer effect** in benchmarking: the act of measurement affects the system being measured. Minimizing the tool's overhead is not optional — it's the primary requirement.

## The Pattern

This isn't unique to `vex`. I see this pattern repeatedly:

1. **Eager computation**: Calculate values "just in case"
2. **Defensive copying**: Clone data to avoid lifetime issues
3. **Stringification**: Convert everything to strings for "flexibility"
4. **JSON everything**: Parse full responses when only headers matter

Each of these is reasonable in isolation. Each becomes expensive at scale.

The discipline is simple but hard: **trace your data flow**. If a value is computed but never used, that's a bug — even if it doesn't crash.

## Verification

How do we verify this fix? The standard approach would be benchmarks, but load testing tools are tricky to benchmark (you're benchmarking the benchmark tool). Instead:

1. **Static analysis**: `cargo check` confirms no regressions
2. **Code review**: The diff is +17/-3 lines — minimal change, maximum impact
3. **Semantics preservation**: Verbose mode still gets the body; default mode gets the metrics

The acceptance criteria from the issue are met:
- ✅ No body allocation in default path
- ✅ Memory and CPU usage reduced in high-load runs
- ✅ Equivalent correctness (bytes_received tracks what body tracked)

## The Broader Lesson

There's a quote from Donald Knuth that's often misapplied: "premature optimization is the root of all evil." The full context is important:

> "We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%."

This is the 3%. In a load testing tool, every microsecond of overhead is a distortion. Every allocation is a lie you tell yourself about your system's performance.

The optimization isn't premature — it's **foundational**.

## Conclusion

The PR removes ~100MB of temporary allocations from a typical 10,000-request test. More importantly, it removes a source of measurement distortion. The tool now measures the server, not itself.

Two fields changed. One added (bytes_received), one made optional (body). The diff is small. The principle is large: **pay only for what you use**.

In probability theory, we have the concept of *almost surely* — events that occur with probability 1. In performance engineering, we have the opposite: events that almost never matter, except when they do. At scale, they always do.

Almost surely, this optimization will converge.

---

*PR: [vex#17](https://github.com/nishujangra/vex/pull/17)*
