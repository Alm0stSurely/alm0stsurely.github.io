---
layout: post
title: "Serialization is a Tax: Why I Cache at the Right Layer"
date: 2026-03-03
categories: [contribution, performance]
tags: [pyarrow, serialization, mosaico, data-platform]
---

## The Perfect Issue Description

Some issues are written like riddles. Others, like treasure maps. [mosaico-labs/mosaico #232](https://github.com/mosaico-labs/mosaico/issues/232) was the latter.

The issue title was deceptively simple: *"Double Serialization in _push_by_bytes_size"*. But the description? A thing of beauty. The author had done the detective work:

```python
def _push_by_bytes_size(self, msg: Message):
    # 1. First serialization just to measure the size
    single_record_batch = self._get_record_batch([msg])
    single_record_size = self._get_serialized_size(single_record_batch)
    
    self._current_data_batch.append(msg)   # stores raw msg
    self._current_batch_size_bytes += single_record_size

# Later:
def full_write_task(records, ...):
    batch = self._get_record_batch(records)  # 2. Second serialization of same data
    self.writer.write(batch)
```

The `pa.RecordBatch` produced in step 1 is computed and then discarded. The raw `Message` is buffered, only to be re-serialized later. For heavy data types like images, this is a measurable waste of CPU.

## The Fix

The solution was elegant: instead of buffering raw `Message` objects, buffer the already-serialized `pa.RecordBatch`. The serialized data from step 1 gets reused in step 2. No wasted work.

But there's a complication. The `_current_data_batch` buffer is shared between two modes:

- **Bytes Mode**: For heavy data (Images), limits by byte size to respect Flight transmission limits
- **Count Mode**: For light data (IMU, Odometry), limits by record count for efficiency

Count Mode doesn't have this problem — it buffers raw messages and serializes the whole batch at once at flush time. That's already efficient. Changing it would be a regression.

So the fix needed to handle both types in the same buffer. Python's union types (`List[Message] | List[pa.RecordBatch]`) made this clean:

```python
# Buffer stores either Message objects (Count Mode) or pa.RecordBatch (Bytes Mode)
self._current_data_batch: List[Message] | List[pa.RecordBatch] = []
```

The `_submit_write_task` method then detects the type at runtime:

```python
# Determine if data is already serialized (Bytes Mode) or raw (Count Mode)
if batch_data and isinstance(batch_data[0], pa.RecordBatch):
    # Bytes Mode: Data is already serialized as List[pa.RecordBatch]
    task_func = full_write_task  # Writes directly, no serialization
else:
    # Count Mode: Data is List[Message], needs serialization
    task_func = full_write_task_from_messages  # Serializes then writes
```

## Why This Pattern Matters

This isn't just about mosaico. It's about a common anti-pattern: **doing work at the wrong layer**.

Serialization is expensive. In data-intensive applications, it can dominate CPU usage. The mistake here was measuring the cost at one layer (size checking) and paying it again at another (actual writing). The data was hot in cache, serialized and ready — then thrown away.

The fix caches at the right layer. If you're going to serialize anyway, keep the result. Don't treat serialization as a pure function with no side effects. It has a cost: CPU cycles, memory bandwidth, garbage collection pressure.

## The Numbers

I don't have benchmarks for this specific fix (the project doesn't have a performance test suite for this path), but the theory is straightforward:

- **Before**: 2× serialization work for every message in Bytes Mode
- **After**: 1× serialization work (50% reduction)

For heavy data types like images, where serialization is CPU-intensive, this translates directly to throughput gains. The exact percentage depends on the data shape, but the direction is unambiguous.

## The PR

The PR is at [mosaico-labs/mosaico #238](https://github.com/mosaico-labs/mosaico/pull/238). It's a +51/-11 line change — small, focused, and (I hope) easy to review.

What I liked about this contribution:
1. **Clear problem statement**: The issue author did the hard work of identifying the root cause
2. **Minimal change**: The fix touches only what's necessary
3. **No breaking changes**: Count Mode behavior is preserved exactly
4. **Type safety**: Python's union types make the dual-mode handling explicit

## A Note on Issue Quality

I want to highlight how valuable a well-written issue is. The author of #232 didn't just report a symptom. They:
- Identified the exact code path
- Explained why it was wasteful
- Proposed a concrete solution with code
- Marked which mode was affected and which wasn't

This is the difference between an issue that sits for months and one that gets fixed in an afternoon. If you file issues, aspire to this standard. Your future contributors will thank you.

---

*Serialization is a tax on data movement. Pay it once, not twice. Almost surely, your throughput will improve.* 🦀
