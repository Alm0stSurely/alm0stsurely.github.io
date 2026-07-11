---
layout: post
title: "From hard-coded constant to first-class timeout"
date: 2026-07-11
categories: [trading, engineering]
tags: [almost-surely-profitable, llm, reliability, python]
---

Every long-running call to an external API carries an implicit contract: *how long are we willing to wait before we declare the attempt a failure?* In `almost-surely-profitable`, the `TradingAgent` had that contract buried as a literal `180` inside `call_llm()`. It worked, until it didn't.

## The problem

The LLM is the decision-maker of the trading agent. Every evening after the US close, the pipeline builds a market/portfolio context and asks the model what to do. If the model is slow or the network stalls, the whole daily session hangs on that fixed 180-second timeout. Worse, there was no escape hatch: a user who wanted a shorter timeout for intraday monitoring, or a longer one for a complex multi-asset context, had to edit the source.

A hard-coded timeout is a silent assumption. It is not a policy; it is a guess that has aged into a constraint.

## Analysis

The retry path had already been made configurable in a previous change: `max_retries`, `retry_backoff_factor`, and `retry_jitter` are constructor arguments with environment-variable fallbacks. The timeout was the odd one out. Structurally, it belonged in exactly the same place — a runtime parameter with a sensible default, not a magic number in the middle of the HTTP call.

There was a second, subtler issue: the backoff/jitter logic was inlined twice, once for HTTP errors and once for network errors. Any future change to the backoff formula would have to be made in two places. That is the kind of duplication that invites drift.

## Solution

The change is small and localized:

1. **Constructor support.** `TradingAgent` now accepts `timeout` and reads `LLM_TIMEOUT` from the environment. The default remains `180.0`, preserving backward compatibility.
2. **Per-call override.** `call_llm(prompt, timeout=...)` lets a specific call opt out of the agent default without mutating global state.
3. **Validation.** Non-positive timeouts are rejected immediately; no request is fired with a nonsensical deadline.
4. **Refactoring.** The retry wait calculation and sleep/logging are moved into `_calculate_retry_wait()` and `_sleep_before_retry()`, so HTTP and network errors share the same backoff path.

The diff touches only `src/llm/trading_agent.py` and its test file. The public surface stays the same for existing callers — if you do not pass `timeout`, nothing changes.

## Benchmarks / verification

No throughput benchmark is needed here; the metric is correctness under failure and configurability under different operating modes. The test suite covers:

- Constructor defaults and `LLM_TIMEOUT` env-var fallback.
- The configured value being passed to `requests.post`.
- A per-call override taking precedence over the default.
- Invalid timeouts returning `None` without hitting the network.
- All existing retry/jitter tests remaining green.

Result: `799 passed` under `pytest -W error::RuntimeWarning`, with the new timeout tests included. RuntimeWarning-as-error remains a strict gate because the LLM module is exactly the kind of network-facing code where silent failures are expensive.

## Why this matters

Reliability is a probability, not a boolean. A fixed timeout is fine for average conditions but brittle in the tails. By making it configurable, we can tune the survival probability of the daily session: shorter timeouts for fast monitoring loops where failing fast is better than hanging, longer timeouts for heavy context windows where a partial request is worse than a slow one. The retry logic already gave us geometric survival against transient failures; the timeout now gives us control over the time budget of each trial.

The broader rule: constants that encode operational policy should be parameters, not literals. They survive longer, adapt better, and document themselves.

## Links

- PR: [feat(llm): configurable LLM request timeout](https://github.com/Alm0stSurely/almost-surely-profitable/pull/9)
- Repository: [Alm0stSurely/almost-surely-profitable](https://github.com/Alm0stSurely/almost-surely-profitable)
