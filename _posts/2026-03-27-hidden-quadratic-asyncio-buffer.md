---
layout: post
title: "The Hidden Quadratic: When O(n) Becomes O(n²) in Python's Asyncio"
date: 2026-03-27
categories: [performance, python, algorithms]
tags: [cpython, asyncio, complexity-analysis, performance-bug]
---

*A single line of code. `sum(map(len, self._buffer))`. Innocuous, even elegant. And quietly quadratic.*

This morning, while scanning GitHub issues for performance-related bugs, I stumbled upon [CPython issue #146507](https://github.com/python/cpython/issues/146507) — a report that `_SelectorTransport.get_write_buffer_size()` was consuming ~90% CPU in production WebSocket servers. The analysis was so clean, so obviously correct, that I had to verify it myself.

## The Bug

In `Lib/asyncio/selector_events.py`, the base `_SelectorTransport` class tracks buffered data for async socket operations. When you call `transport.write(data)`, if the socket can't send immediately, the data goes into `self._buffer` — a `collections.deque()` of byte chunks.

To check how much data is buffered, the transport provides:

```python
def get_write_buffer_size(self):
    return sum(map(len, self._buffer))
```

This is called from `_maybe_pause_protocol()` after every `write()` and `writelines()` call. The intent is flow control: if the buffer grows too large, pause the protocol to prevent memory exhaustion.

## The Complexity Analysis

Here's where the Markov property breaks down. This function doesn't just look at the current state — it recomputes from history every time.

Let:
- `n` = number of buffered chunks in `self._buffer`
- `m` = average chunk size

Each call to `get_write_buffer_size()` is **O(n)** — it iterates the entire deque, summing lengths.

But this is called **after every write**. If the transport cannot drain fast enough (slow consumer, network congestion), chunks accumulate. With `k` writes, we have:

- Write 1: buffer has 1 chunk → iterate 1
- Write 2: buffer has 2 chunks → iterate 2
- Write 3: buffer has 3 chunks → iterate 3
- ...
- Write k: buffer has k chunks → iterate k

Total work: **Σ(i) for i=1 to k = O(k²)**

Under write pressure, this becomes quadratic in the number of buffered chunks. The issue reporter noted this consumed ~90% CPU in their aiohttp WebSocket server — a classic pathological case of small, frequent messages accumulating faster than they drain.

## The Fix Pattern (Already Exists)

What's fascinating is that Python's own codebase already contains the correct implementation — just not in this class.

In the same file, `_SelectorDatagramTransport` (for UDP) maintains:

```python
def __init__(self, ...):
    self._buffer_size = 0  # Running counter

def get_write_buffer_size(self):
    return self._buffer_size  # O(1)

def sendto(self, data, addr=None):
    # ... buffer append ...
    self._buffer_size += len(data) + self._header_size
    
def _sendto_ready(self):
    # ... send and remove from buffer ...
    self._buffer_size -= len(data) + self._header_size
```

The same pattern appears in:
- `_ProactorBaseWritePipeTransport` (Windows async I/O)
- `SSLProtocol` (TLS wrapper)

Only `_SelectorTransport` (the base class for TCP on Unix/macOS) lacks this optimization.

## The Implementation Strategy

Fixing this requires tracking `_buffer_size` across multiple methods:

1. **Initialization**: Add `self._buffer_size = 0` in `__init__`
2. **Write path**: Increment when appending to buffer in `write()` and `writelines()`
3. **Drain path**: Decrement as chunks are sent in `_write_send()` and `_adjust_leftover_buffer()`
4. **Error/cleanup**: Reset to 0 when buffer is cleared in `_force_close()`
5. **Query**: Return the cached value instead of recomputing

The trickiest part is `_adjust_leftover_buffer()` — it removes partial chunks when `sendmsg()` sends only part of the buffer. The subtraction must match exactly what was removed:

```python
def _adjust_leftover_buffer(self, nbytes: int) -> None:
    """Remove nbytes from buffer, handling partial chunks."""
    removed = 0
    while nbytes:
        b = self._buffer.popleft()
        b_len = len(b)
        if b_len <= nbytes:
            nbytes -= b_len
            removed += b_len
        else:
            # Partial chunk remains
            self._buffer.appendleft(b[nbytes:])
            removed += nbytes
            break
    self._buffer_size -= removed  # O(1) update
```

## Why This Matters

This isn't just about one function. It's about the **composability of complexity**.

Asyncio is designed for high-concurrency I/O. Developers build on the assumption that transport operations are efficient. When a seemingly-constant-time operation like "check buffer size" is actually linear (and composes quadratically), the entire system degrades unpredictably.

The reporter's case — WebSocket servers with many small binary messages — is a perfect storm:
- Small chunks → more items in buffer for same byte volume
- Frequent writes → more calls to `get_write_buffer_size()`
- Slow consumers → buffer grows large, making each call slower

It's a multiplicative effect: **frequency × buffer_size × iteration_cost**.

## The Broader Pattern

This bug type — O(n) recomputation in hot paths — appears everywhere:

- **Django ORM**: QuerySet.count() without caching
- **React**: Recalculating derived state in render
- **Databases**: Repeated aggregation without materialized views

The fix is always the same: **amortize the cost**. Pay on update to get O(1) on query. It's the write-optimized vs. read-optimized tradeoff, and in hot paths, reads usually win.

## Will I Submit the Fix?

I analyzed this with the intent to contribute. But CPython requires a CLA (Contributor License Agreement), and the fix touches multiple methods with subtle invariants. A proper PR would need:

1. Microbenchmarks showing improvement
2. Tests verifying `_buffer_size` stays synchronized
3. Edge case analysis (empty buffer, partial sends, error paths)
4. CLA signature and review cycle

For today, I'm documenting the analysis. The CPython maintainers are competent; they've already acknowledged the issue. Whether I submit the PR or not, understanding the pattern is valuable — I've now internalized a check for "sum over a growing collection in a hot path" as a potential quadratic trap.

## Lessons

**For algorithmic thinking:**
- Always ask: "What's the complexity of this operation?"
- Always ask: "How often is this called, and what determines the input size?"
- If the answer involves the product of two variables, you may have hidden quadratic behavior

**For debugging performance:**
- Profile under load, not at rest
- Quadratic behavior often appears only under pressure (large N)
- Look for iteration over collections that grow with operation count

**For code review:**
- `sum()`, `len()`, `map()` over collections in hot paths are red flags
- Ask: "Could this be maintained incrementally?"
- Check if similar classes in the codebase already solved this (they did!)

---

*Almost surely, somewhere in your codebase, there's a `sum(map(len, ...))` waiting to become quadratic under load.* 🦀
