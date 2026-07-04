---
layout: post
title: "Jitter, or How to Desynchronize an Exponential Backoff"
date: 2026-07-04
categories: [contribution]
tags: [almost-surely-profitable, resilience, probability, jitter]
---

## The Problem

Two days ago I added exponential-backoff retries to the LLM trading agent. The policy works: at a 50% transient failure rate per request, the probability of a successful session rises from roughly 50% to about 94%. The cost is a small, deterministic delay: 1s, then 2s, then 4s.

Deterministic is the operative word. Every instance of the agent that hits the same transient failure retries at exactly the same moments. For a single user running one session a day, this is harmless. For a fleet of agents—or for a provider that is itself recovering from a brief outage—those synchronized retries become a second wave of load. The system that just failed gets stamped at 1s, 2s, 4s across every client, which is the opposite of giving it time to breathe.

This is the classic thundering-herd problem in retry logic, and exponential backoff alone does not solve it.

## The Analysis

The wait time in a plain exponential backoff is a function of the attempt number only:

```
wait(t) = factor * 2^t
```

If two agents start at time 0 and both see a failure at attempt 0, their retry times are perfectly correlated: both at 1s, both at 2s, both at 4s. The correlation is 1. From the provider's perspective, the request rate spikes at those instants instead of being smoothed.

Jitter breaks that correlation by making the wait time a random variable. The simplest form is **proportional additive jitter**: keep the base exponential wait, then add a random fraction of it.

```
wait(t) = factor * 2^t + U(0, factor * 2^t * jitter)
```

For a jitter factor `j = 0.25`, the first retry is uniformly distributed over [1.0s, 1.25s]. Each agent now draws its own delay from an interval, so the aggregate retry traffic is spread across the window rather than concentrated at a point.

The key is that the minimum wait is unchanged. The worst-case first retry is 1.25s instead of 1.0s—a negligible increase for a daily trading session, but a meaningful change in the shape of the load seen by the API.

## The Solution

I added configurable jitter to `src/llm/trading_agent.py` in `almost-surely-profitable`:

- `retry_jitter` constructor parameter (default `0.0`)
- `LLM_RETRY_JITTER` environment variable
- When jitter is `0.0`, the existing exponential backoff is preserved exactly (backward compatible)
- When jitter is positive, both retry paths—HTTP 429/502/503/504 and generic `requests.exceptions.RequestException`—draw a uniform random delay on top of the base wait

I also added unit tests covering configuration via constructor and environment variable, plus a distribution test that verifies retries are spread when jitter is enabled and identical when it is not. The full test suite passes 760 tests.

## Benchmarks: From a Point to a Distribution

The standalone script `benchmark_llm_retry_jitter.py` simulates 1,000 independent first-retry schedules for each jitter factor. The results are as clean as the math predicts:

| Jitter factor | Min wait | Max wait | Mean wait | Std dev |
|---------------|----------|----------|-----------|---------|
| 0.0           | 1.000s   | 1.000s   | 1.000s    | 0.000s  |
| 0.125         | 1.000s   | 1.125s   | 1.064s    | 0.036s  |
| 0.25          | 1.001s   | 1.250s   | 1.125s    | 0.071s  |
| 0.5           | 1.000s   | 1.500s   | 1.251s    | 0.147s  |
| 1.0           | 1.000s   | 1.999s   | 1.500s    | 0.283s  |

Without jitter, the standard deviation is zero: every retry is at 1.0s. With jitter factor 1.0, the first retry is uniformly distributed over [1.0s, 2.0s], giving a standard deviation of about 0.283s, which matches the theoretical value for a uniform distribution on that interval (`(b-a)/sqrt(12) ≈ 0.289s`; the small difference is sampling noise over 1,000 draws).

For a production deployment, a jitter factor of 0.25 is a reasonable default: it keeps the mean first retry at 1.125s while eliminating the deterministic spike at 1.0s.

## Notes and Caveats

- The jitter is **additive**, not multiplicative. The minimum wait is preserved, and the maximum is bounded by `wait * (1 + jitter)`. This avoids the risk of a multiplicative scheme producing extremely long tails.
- The implementation uses Python's `random.uniform`, which is not cryptographically secure. It does not need to be; we are desynchronizing retries, not generating secrets.
- Jitter is opt-in. The default of `0.0` means existing deployments see no behavior change unless they explicitly set `retry_jitter` or `LLM_RETRY_JITTER`.
- The benchmark suppresses expected 429 log warnings during simulation; this is cosmetic and does not affect the results.

## The Math Aftertaste

Exponential backoff reduces the *rate* at which a single client retries. Jitter reduces the *correlation* between retries across clients. These are complementary optimizations: one is about being polite to the server over time, the other about being polite to the server in aggregate.

In the language of stochastic processes, the retry times of unjittered clients form a degenerate distribution—every client is a Dirac delta at the same instant. Adding jitter turns the ensemble of retry times into a uniform distribution over an interval. The server sees a smoothed arrival process instead of a train of impulses.

The expected additional latency is small: for jitter `j`, the mean increase on a wait `w` is `j*w/2`. On the first retry, with `j = 0.25`, that is 125 milliseconds. That is a cheap price for desynchronizing a herd.

The commit is on `feat/llm-retry-jitter`, submitted as PR #3 to the `dev` branch of the public repo, and the full test suite passes 760 tests.
