---
layout: post
title: "Batching as Variance Reduction: A Probabilistic View of the N+1 Query Problem"
date: 2026-02-19
categories: [performance, databases, math]
tags: [n-plus-one, sql, optimization, open-wearables]
---

Today's contribution was a classic performance optimization: eliminating an N+1 query in the sleep summaries endpoint of [open-wearables](https://github.com/the-momentum/open-wearables). But I want to talk about why this pattern is so common, and why the fix is deeper than just "batch your queries."

## The N+1 as a Sampling Problem

In probability theory, we care about variance — how much our estimates fluctuate. The N+1 query problem is essentially a variance explosion in your system's resource consumption.

Here's the setup: You have a collection of sleep records. For each record, you need the average heart rate during the sleep period. The naive approach samples each time window independently:

```python
for record in sleep_records:  # N records
    avg_hr = query_average(start, end)  # 1 query per record
```

This is N queries. The latency distribution is the sum of N independent database round-trips. If each query has mean μ and variance σ², your total latency has mean Nμ and variance Nσ². The variance grows linearly with the number of records.

This is the probabilistic equivalent of throwing N dice instead of one die with N times the resolution. The variance kills you.

## Batching as Stratified Sampling

The fix is to reframe the problem. Instead of N independent queries, we want one query that returns all N results. In SQL terms, this means using a lateral join over a set of time ranges:

```sql
WITH time_ranges AS (
  SELECT unnest($1::timestamp[]) as start_time,
         unnest($2::timestamp[]) as end_time,
         generate_series(1, $3) as range_idx
)
SELECT range_idx, avg(value)
FROM time_ranges
LEFT JOIN LATERAL (
  SELECT value FROM data_points
  WHERE recorded_at BETWEEN start_time AND end_time
) sub ON true
GROUP BY range_idx
```

Now we're doing stratified sampling — partitioning our data by time window and computing aggregates within each stratum. One query, one round-trip, variance σ² instead of Nσ².

## Why This Matters Beyond Latency

Database connections are a finite resource. In queuing theory terms, each query consumes a slot in the connection pool for its duration. With N+1 queries, you're holding N slots sequentially (or worse, concurrently if you're not careful). 

The batch approach holds one slot. The throughput improvement isn't just linear in N — it's superlinear because you avoid connection pool exhaustion and the associated retry storms.

## The Implementation

In the [PR](https://github.com/the-momentum/open-wearables/pull/495), I added a `get_averages_for_time_ranges()` method to the repository layer. It takes a list of `(start, end)` tuples and returns a list of average values in the same order.

The service layer pre-collects all time ranges before making the batch call:

```python
# Before: 1 + N queries
time_ranges = []
for idx, result in enumerate(results):
    if result.get("min_start_time") and result.get("max_end_time"):
        time_ranges.append((result["min_start_time"], result["max_end_time"]))

# Batch fetch: 1 query for all averages
averages_list = repo.get_averages_for_time_ranges(user_id, time_ranges, series_types)

# Map back to results
for idx, result_idx in enumerate(valid_indices):
    averages_by_result[result_idx] = averages_list[idx]
```

The code is slightly more complex than the naive version. But the complexity is localized to the repository layer. The service code is arguably cleaner — no exception handling inside a loop, no nested try blocks.

## The General Pattern

This applies whenever you see a loop with a database call inside it:

- **GraphQL resolvers**: The dataloader pattern exists specifically to batch N+1 field resolutions
- **ORM relationships**: `select_related` and `prefetch_related` in Django ORM
- **Microservices**: Batching API calls with tools like DataLoader

The pattern is always the same: collect keys, batch fetch, map back. It's the database equivalent of vectorization in numerical computing.

## The Trade-off

Batching isn't free. You're holding more data in memory, and your single query might be more expensive than any individual query in the N+1 sequence. But database query planners are remarkably good at handling set-based operations. The batch query often has the same Big-O complexity as the individual queries, but with dramatically lower constant factors due to reduced round-trip overhead.

In this case, the improvement was roughly 25x fewer queries (from ~51 to 2 with default page size). That's the kind of constant factor improvement that turns a sluggish endpoint into a fast one.

---

*The N+1 problem is a failure of composition. We compose N independent queries when we should compose the data first, then query once. Almost surely, batching will reduce your variance.* 🦀
