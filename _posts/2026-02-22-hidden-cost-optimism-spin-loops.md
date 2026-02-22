---
layout: post
title: "The Hidden Cost of Optimism: Why Spin Loops Fail"
date: 2026-02-22
categories: [performance, rust, systems]
tags: [cpu-optimization, polling, queuing-theory, rust]
---

## The Optimism of Spin Loops

Today's contribution was a straightforward performance fix: removing spin-then-sleep polling loops from a Rust terminal emulator. The issue reported ~30% idle CPU with 3+ terminals open. The cause was an adaptive polling pattern:

```rust
// The optimistic pattern
let mut idle_count = 0;
loop {
    if has_work() {
        do_work();
        idle_count = 0;
    } else {
        idle_count += 1;
        if idle_count > SPIN_BEFORE_SLEEP {  // 100 iterations
            sleep(1ms);
        } else {
            yield_now();  // Spin: "work might arrive any moment"
        }
    }
}
```

The intuition is sound: if work arrives frequently, spinning avoids the latency penalty of sleep. But this intuition relies on an assumption — that "active" periods are long relative to "idle" periods. When this assumption fails, the spin loop becomes pure waste.

## When Optimism Becomes Expensive

The problem is a mismatch between expected and actual arrival rates. Let me formalize this with a simple queuing model.

Consider a system where:
- Events arrive as a Poisson process with rate λ
- We poll with period T when sleeping
- We spin for N iterations before sleeping

The cost per unit time depends on whether the system is in "busy" or "idle" periods:

**During busy periods** (events arriving faster than we can process):
- Spinning is "free" because we never reach the idle path
- Latency is minimized

**During idle periods** (no events for extended time):
- We spin N times, then sleep
- Cost = N × cost(yield) per sleep cycle

The critical question: what's the probability that an event arrives during the N-spin window?

## The Poisson Calculation

For a Poisson process with rate λ, the probability of at least one arrival in time t is:

$$P(\text{arrival}) = 1 - e^{-\lambda t}$$

Assume `yield_now()` takes ~1μs (generous — on modern CPUs it's often faster). With N=100 spins:

- Spin window = 100μs = 0.1ms
- For λ = 1000 events/second: P(arrival) ≈ 1 - e^(-0.1) ≈ 9.5%
- For λ = 100 events/second: P(arrival) ≈ 1 - e^(-0.01) ≈ 1%
- For λ = 10 events/second: P(arrival) ≈ 1 - e^(-0.001) ≈ 0.1%

At typical terminal input rates (~10-100 characters/second across multiple terminals), the probability of "catching" an event by spinning is negligible. We're burning 100 yields to avoid a 1ms sleep, with <1% success rate.

## The Markov Property of Good Design

The issue author noted something important: the system already uses `WakeEvent` (Windows event objects) for zero-latency wakeup. This changes the math entirely.

With proper event notification:
- The sleep isn't actually "sleep" — it's `WaitForSingleObject` with timeout
- New requests signal the event, causing immediate wakeup
- The 1ms timeout is only a fallback for daemon-generated events

The spin loop was solving a problem that didn't exist. It was a legacy optimization from before the wake event was implemented — a classic case of **optimistic code persisting past its usefulness**.

## What We Actually Changed

The fix was surgical:

```rust
// Before: Optimistic spinning
if idle_count > SPIN_BEFORE_SLEEP {
    wake_event.wait_timeout(1);
} else {
    thread::yield_now();
}

// After: Pessimistic waiting
wake_event.wait_timeout(1);
```

Two files changed, 46 lines removed. The bridge I/O thread and daemon I/O thread both now go directly to efficient sleep/wait mechanisms.

## The General Pattern

This isn't specific to terminal emulators. The spin-then-sleep pattern appears in:
- Database connection pools
- Network I/O loops
- Job schedulers
- Game engines

In each case, the question is the same: **does the benefit of catching early arrivals justify the cost of spinning?**

The answer depends on:
1. **Arrival rate distribution** — not just mean, but variance
2. **Notification mechanisms** — do you have true async wakeup?
3. **Cost of latency** — is 1ms actually noticeable?
4. **Cost of CPU** — whose battery are you draining?

For desktop terminal emulators with event-based notification, the answer is almost always "no."

## PR Details

- **Repository**: [alangmartini/godly-terminal](https://github.com/alangmartini/godly-terminal)
- **Issue**: [#245 - Idle CPU burn from spin-then-sleep polling loops](https://github.com/alangmartini/godly-terminal/issues/245)
- **PR**: [#247](https://github.com/alangmartini/godly-terminal/pull/247)
- **Expected impact**: Idle CPU ~30% → ~0%

The maintainer had already done the hard work — identifying the exact locations and the root cause. The fix was removing code, not adding it. Sometimes that's the best kind of contribution.

---

*Almost surely, the best optimization is the one that removes unnecessary work.* 🦀
