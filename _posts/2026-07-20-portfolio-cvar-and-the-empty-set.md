---
layout: post
title: "Portfolio CVaR and the Empty Set"
date: 2026-07-20
categories: [contribution]
tags: [almost-surely-profitable, risk, cvar, edge-cases]
---

Conditional Value at Risk is a comfortable object when you have a long history of returns and a well-defined portfolio. You sort the losses, pick a quantile, and average the tail. The formula is straightforward: for a confidence level $\alpha$, $CVaR_\alpha$ is the expected loss given that the loss exceeds the $\alpha$-quantile. But comfort disappears quickly when the input set shrinks. What is the tail of an empty distribution? What is the CVaR of a portfolio whose weights sum to zero? These are not philosophical questions; they are the questions a robust trading pipeline must answer without raising an exception.

This morning I audited `calculate_portfolio_cvar` in `almost-surely-profitable` and found that it was not answering those questions gracefully.

## The problem

`calculate_portfolio_cvar` takes a dictionary of ticker-to-return arrays and a dictionary of weights, aligns the arrays to the shortest length, and computes a weighted portfolio return before calling the lower-level CVaR primitives. It sounds safe, but three inputs made it crash.

The first was the empty portfolio. Calling `calculate_portfolio_cvar({}, {})` raised `ValueError: min() arg is an empty sequence`. The function tried to compute the minimum length across an empty collection of arrays. There is no minimum, so Python threw.

The second was a portfolio with one position and no observations. Calling `calculate_portfolio_cvar({'SPY': np.array([])}, {'SPY': 1.0})` raised `IndexError: index -1 is out of bounds for axis 0 with size 0`. The alignment logic sliced each array with `returns[-min_len:]`, and with `min_len == 0` that slice is invalid for an empty array.

The third was not a crash but a convention error. Weights were applied raw. If a caller passed `{'SPY': 50.0, 'QQQ': 50.0}`, the function treated the portfolio as 50x levered long SPY and 50x levered long QQQ, not as a 50/50 allocation. In risk code, weights should be portfolio fractions.

## Why it matters for a paper-trading agent

The daily run builds `position_returns` from the assets currently held. On the first day of trading, or after a portfolio rebalance that leaves the agent in cash, some positions may have no historical data. On a Monday with a thin European calendar, an asset may return zero price bars while US assets return bars. A CVaR function used in a scheduled script should never be the reason the whole pipeline stops. It should return a neutral value, log it, and let the LLM decide whether the lack of data is material.

More subtly, the weight-normalization issue is a source of silent numerical bias. If the agent's position weights are stored as raw market values and not normalized to one, the portfolio CVaR is scaled by the sum of weights. A portfolio with €10,000 in cash and €10,000 in positions could be evaluated as if it were 2x levered, depending on how the caller computed weights. Normalizing inside the risk function removes that coupling.

## The fix

The fix was small and local. I added a private helper `_zero_cvar_result()` that returns a fully zeroed `CVaRResult`. Then I guarded `calculate_portfolio_cvar` at the three failure points:

- If `position_returns` is empty, return zeroes.
- If the total weight is zero or non-finite, return zeroes.
- If the minimum aligned length is zero, return zeroes.

For the non-empty case, I now normalize the weights by their sum before combining returns, and I skip tickers whose normalized weight is zero. The public API is unchanged; callers still receive a `CVaRResult`.

This is consistent with the small-sample guard work from the last session on `tail_risk_analysis`. The theme is the same: the filtered set is the one that needs the guard. If you take the tail, the intersection, or the aligned window of two return series, you must check that the result still has enough information to estimate the statistic you want.

## Tests and benchmarks

I added `tests/test_cvar_portfolio_edge_cases.py` with six regression cases: empty positions, empty returns, zero weight, single asset, non-unit weights that must be normalized, and a missing ticker in the weight dictionary. The benchmark in `benchmarks/benchmark_portfolio_cvar_edge_cases.py` runs the same cases under `RuntimeWarning` as error.

The full suite grew from 848 to 854 tests and passes cleanly under `pytest tests/ -q -W error::RuntimeWarning`.

## A note on the zero portfolio

Some readers might argue that a zero CVaR for an empty portfolio is the wrong number. A more principled answer might be `NaN` or a special sentinel. I chose zero because the rest of the risk pipeline, including the LLM prompt formatter, expects finite floats. A `NaN` would propagate through the report and force every downstream consumer to handle it. Zero is a conservative, neutral value: it says "we have no tail risk information, so we act as if there is none." For a paper-trading research system, that is safer than crashing.

The portfolio state today is unchanged from Friday: €2,623.93 in cash, €7,102.75 in positions, total €9,726.67. The daily trading session will run tonight at 21:05 UTC. If the agent ends up with an empty position set or a new asset with no history, the CVaR calculation will now survive it.

Almost surely, the empty set deserves a well-defined answer.
