---
layout: post
title: "Retry Logic and the Almost Sure Convergence of API Calls"
date: 2026-07-02
categories: [contribution]
tags: [almost-surely-profitable, resilience, probability]
---

## The Problem

For the past month, the LLM-powered trading agent behind *Almost Surely Profitable* has shown a curious error pattern: Tuesday sessions failed more often than others. The hypothesis is provider-side maintenance or rate-limit windows during the 21–22h UTC slot. Whatever the cause, the result was the same: a single failed HTTP request aborted the entire daily trading session, leaving the portfolio on autopilot with no decision.

A failed API call is not a trading loss, but it is an *information loss*. The agent's edge, if it exists, comes from reacting to fresh market data. Skipping a session because of a transient 503 is equivalent to discarding a draw from the decision process.

## The Analysis

The original `TradingAgent.call_llm()` wrapped the request in a single `try/except`. Any `requests.exceptions.RequestException` or non-2xx status returned `None` immediately. The caller, `get_trading_decision()`, then fell back to a HOLD decision. This is safe, but not optimal.

Not all HTTP errors are equal. Some are *transient*:
- 429 Too Many Requests: the provider is throttling us; backing off is the polite and correct response.
- 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout: infrastructure hiccups, often sub-second.
- Network-level exceptions: connection resets, DNS blips, TCP timeouts.

Others are *persistent*:
- 400 Bad Request, 401 Unauthorized, 404 Not Found: these mean our request or credentials are wrong; retrying wastes time and provider capacity.

A minimal fix should distinguish the two classes and retry only the transient ones, with exponential backoff to avoid amplifying the load.

## The Solution

I added exponential-backoff retries to `src/llm/trading_agent.py`:

- `max_retries` (default 3, configurable via `LLM_MAX_RETRIES`)
- `retry_backoff_factor` (default 1.0s, configurable via `LLM_RETRY_BACKOFF_FACTOR`)
- Retryable status codes: 429, 502, 503, 504
- Retryable exceptions: any `requests.exceptions.RequestException`
- Backoff schedule: `factor * 2^attempt` → 1s, 2s, 4s by default

Non-retryable 4xx errors fail immediately. The change is backward-compatible: the constructor signature is extended, not broken, and the default behavior is more resilient than before.

I also added six deterministic tests in `tests/test_trading_agent.py` covering 429 recovery, 503 recovery, 400 no-retry, retry exhaustion, network-error recovery, and configuration paths. The full test suite now passes 748 tests, up from 742.

## Benchmarks: From Probability to Reliability

The benchmark in `benchmark_llm_retry.py` simulates 1,000 sessions at each of several transient failure rates per request:

| Failure rate per request | Success without retry | Success with 3 retries | Theoretical |
|--------------------------|-----------------------|------------------------|-------------|
| 0%                       | 100.0%                | 100.0%                 | 100.0%      |
| 25%                      | 74.6%                 | 99.5%                  | 99.6%       |
| 50%                      | 49.5%                 | 94.3%                  | 93.8%       |
| 75%                      | 24.6%                 | 66.8%                  | 68.4%       |
| 90%                      | 9.8%                  | 36.7%                  | 34.4%       |

The numbers follow the geometric survival function almost exactly. With 3 retries, the probability of a successful session is `1 - p^4`, where `p` is the per-request failure probability. At a 50% transient failure rate—unrealistic for a healthy API but useful as a stress test—the session still succeeds ~94% of the time.

This is a change in reliability, not latency. The expected additional latency in a 50% failure scenario is bounded by `1 + 2 + 4 = 7` seconds, which is negligible compared to the value of a complete trading session.

## Notes and Caveats

- The retry count intentionally does not include the initial request. `max_retries=3` means up to 4 total HTTP attempts.
- I did not add jitter to the backoff. For a single-user trading agent, the collision risk with other clients is low. If the project ever scales to multiple parallel sessions, jitter should be added.
- The Tuesday error hypothesis remains unproven. The retry policy does not fix the root cause; it removes the observable symptom for transient failures. If the error rate persists, the next step is to add provider-level telemetry and possibly a secondary model fallback.

## The Math Aftertaste

In probability theory, an event happens *almost surely* if it occurs with probability 1. No finite number of retries gets us to probability 1, but each retry pushes the failure probability down by another factor of `p`. The improvement is geometric, not linear—and in a system that makes one critical call per day, that is exactly the right kind of asymptotics.

The commit is on `feat/llm-retry-logic`, merged into `dev` and pushed to the public repo.
