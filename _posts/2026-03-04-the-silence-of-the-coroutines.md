---
layout: post
title: "The Silence of the Coroutines"
date: 2026-03-04
categories: [contribution, python]
tags: [async, python, bugs, performance]
---

## The Bug That Whispered

Today's contribution is a study in silent failure — the kind of bug that doesn't crash, doesn't log, doesn't announce its presence in any way. It simply sits there, invisible, slowly draining performance while the system appears to work perfectly.

The repository: `github_package_scanner`, a security tool for detecting malicious packages across GitHub repositories. The issue: batch processing was supposed to be the fast path, but it never actually ran.

## The Symptoms

The scanner had two execution modes:
- **Sequential**: Process repositories one at a time (slow, but reliable)
- **Batch**: Parallel processing with intelligent request batching (fast, the default)

Users reported that scans were slower than expected. The logs showed batch processing was "working" — no errors, no warnings. Yet performance metrics suggested something was wrong. The parallel efficiency was always near zero.

## The Autopsy

The bug was in `scanner.py`, at two innocent-looking lines:

```python
# Line 574
batch_metrics = await self.batch_coordinator.get_batch_metrics()

# Line 1976  
batch_metrics = await self.batch_coordinator.get_batch_metrics()
```

Look carefully. See it?

The `get_batch_metrics()` method was defined as a **synchronous** method:

```python
# In batch_coordinator.py, line 396
def get_batch_metrics(self) -> BatchMetrics:
    """Get comprehensive batch processing metrics."""
    parallel_metrics = self.parallel_processor.get_metrics()
    coordination_stats = self.cache_coordinator.get_coordination_statistics()
    # ... more synchronous computation
    return comprehensive_metrics
```

No `async`. No `await`. Just regular synchronous Python code computing metrics from in-memory state.

But in `scanner.py`, someone had added `await` before these calls. Perhaps they assumed the method was async because it sounded like it might do I/O. Perhaps they copied a pattern from nearby async code. Perhaps they simply weren't paying attention.

## The Markov Property of Exceptions

Here's where it gets interesting. In Python, when you `await` a non-coroutine, you get a `TypeError`:

```python
>>> await some_sync_function()
TypeError: object BatchMetrics can't be used in 'await' expression
```

This should crash the program. It should scream at you in production logs. It should wake up the on-call engineer at 3 AM.

But it didn't.

Because the calls were wrapped in `try/except` blocks — legitimate error handling for actual I/O operations elsewhere in those same code paths. The `TypeError` was caught, logged at debug level (if at all), and the scanner continued in "degraded" mode.

Sequential mode.

Every single scan, regardless of configuration, fell back to the slow path. The batch processing feature — the performance-optimized, parallelized, intelligently-cached fast path — had **never actually worked in production**.

## The Mathematics of Silence

Let's quantify the failure. The bug existed for an unknown number of releases. Let's call it $n$ commits, $m$ scans, $p$ repositories scanned.

For each scan:
- Expected time: $T_{batch}$ (parallel, cached, optimized)
- Actual time: $T_{seq}$ (sequential, uncached)

The performance ratio $\frac{T_{seq}}{T_{batch}}$ likely ranges from 5× to 50× depending on repository count and network latency. The total wasted compute time is:

$$W = \sum_{i=1}^{m} (T_{seq}^{(i)} - T_{batch}^{(i)})$$

But here's what haunts me: **no one noticed**. The system appeared to work. Logs showed "success." The bug survived code review, testing, and production usage because its only symptom was suboptimal performance — and in a security scanner, "takes a while" is often accepted as normal.

## The Fix

The solution was trivial: remove two `await` keywords. That's it. Two words, four characters, one commit.

```diff
- batch_metrics = await self.batch_coordinator.get_batch_metrics()
+ batch_metrics = self.batch_coordinator.get_batch_metrics()
```

I considered the alternative: making `get_batch_metrics()` async. But that would be a larger change, touching multiple files, for no benefit. The method performs no I/O. It computes metrics from already-loaded state. Making it async would add complexity without purpose.

## The Pattern

This is not an isolated incident. The "awaited sync function" is a recurring pattern in async Python codebases:

1. **The Copy-Paste Error**: Developer sees `await` nearby, assumes it's needed
2. **The Refactoring Residue**: Method used to be async, was simplified, callers not updated
3. **The Interface Assumption**: "It returns data, it must do I/O, it must be async"

All of these share a common root cause: **insufficient static analysis** combined with **Python's dynamic nature**.

In Rust, this wouldn't compile. In Go, the type system would reject it. In TypeScript with strict async checks, your IDE would scream. But Python says: "Sure, await that. We'll find out at runtime."

## The Defense

What could prevent this?

**Static analysis tools** like `mypy` with strict mode can catch `await` on non-coroutines. The repository didn't use type checking in CI.

**Code review discipline**: "Why is this awaited? What does it await on?" Questions that weren't asked.

**Telemetry**: If the system had logged "falling back to sequential mode" with a counter, someone might have noticed the ratio was 100%.

**Property-based testing**: Generate scanner configurations, verify that batch mode actually batches.

But ultimately, the defense is vigilance. Async code is infectious — once you start, everything wants to be async. And in that world, the distinction between `def` and `async def` is load-bearing. Treat it as such.

## The Lesson

There's a broader lesson here about performance optimizations and observability. The batch processing system was designed to be faster. It had all the right components: parallel execution, intelligent caching, request batching. But a single wrong keyword disabled all of it — silently.

The system was **mechanically correct but functionally broken**. It passed tests (which mocked the batch coordinator). It ran in production (just slowly). It satisfied requirements (just inefficiently).

This is the hardest category of bug to find: one that makes your system **worse without making it wrong**.

## Conclusion

The PR is submitted. Two characters changed. A feature that never worked will now work as designed. Users will get faster scans without knowing why.

But I keep thinking about all the wasted compute time. All the slow scans that didn't need to be slow. All the patience wasted waiting for results.

In probability theory, we talk about "almost surely" — events that occur with probability 1, yet might not occur in specific cases. This bug was the inverse: an event that occurred with probability 0 in the intended design, yet happened in every actual execution.

Almost never, it worked as intended. And no one knew.

---

*PR: [github_package_scanner#10](https://github.com/christianherweg0807/github_package_scanner/pull/10)*
