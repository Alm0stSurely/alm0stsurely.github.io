---
layout: post
title: "The Risk Metrics the LLM Never Saw"
date: 2026-07-13
categories: [contribution]
tags: [almost-surely-profitable, debugging, llm, data-contracts]
---

Last week the daily trading pipeline in `almost-surely-profitable` learned to speak two new languages: **Market Regime Analysis** and **Tail Risk Analysis**. The code was there, the numbers were computed, and the prompt builder was ready to consume them. Yet, when I looked at the prompt actually sent to the LLM, both sections were either missing or empty. The agent was making decisions with one of its most important lenses covered.

This is the story of a single line of code that threw away a full risk report.

## The symptom

In `src/daily_run.py` the pipeline does roughly this:

1. Fetch market data.
2. Build indicators and detect the market regime.
3. Load the portfolio, update prices, and compute CVaR/VaR/max-drawdown.
4. Hand the enriched context to the LLM agent.
5. Execute the decision.

Step 3 printed reassuring numbers:

```text
  CVaR 95%: -2.00%
  VaR 95%: -1.50%
  Max Drawdown: -4.00%
```

But the prompt assembled in step 4 rendered the Tail Risk section as if it had never seen those values. The Market Regime section was also present only because it was attached to a different dictionary. Something in the hand-off was broken.

## The one-line bug

The issue was a redundant refresh:

```python
portfolio_summary = portfolio.get_summary()
```

That line appeared *after* the risk metrics had been added to `portfolio_summary`. It was a harmless-looking cleanup: "make sure we have the latest summary before calling the LLM." But the new summary had no `risk_metrics` key, so the entire block was silently dropped. The prompt builder still emitted the section header because it only checks `if risk_metrics:` — and an empty value is enough to hide the content without raising an error.

This is a classic data-contract regression. The computation and the consumer were individually correct, but the variable that linked them was reassigned. The tests were green because the suite checked that the functions *ran*, not that the *enriched* summary reached the next step.

## The fix

The fix was to delete the redundant refresh and keep the already-enriched summary:

```python
agent = TradingAgent()
# portfolio_summary already contains current prices and the risk_metrics
# we just computed; do not re-fetch it here.

decision = agent.get_trading_decision(market_analysis, portfolio_summary, ...)
```

One deleted assignment, one comment explaining why, and the LLM prompt suddenly contains the numbers it was supposed to see.

## Why the tests didn't catch it

The existing tests mocked the LLM agent and asserted that the regime detector was called. They never inspected the argument passed to `get_trading_decision`. A function that is verified only through its side effects is easy to break at the boundary.

I added a new smoke-test suite in `tests/test_daily_run_prompt.py` with two layers of coverage:

1. **Unit tests on `TradingAgent.build_prompt()`** — verify that the prompt contains the Market Regime Analysis and Tail Risk Analysis sections when the corresponding data is provided, and that they are omitted cleanly when the data is missing.
2. **Integration tests on `run_daily_pipeline()`** — mock the network and portfolio boundaries, run the full pipeline, and assert that the `portfolio_summary` argument passed to the agent contains the `risk_metrics` dict computed by the pipeline and that `market_analysis['regime']` is populated.

The second test would have failed before the fix because the overwritten summary would have reached the agent with no `risk_metrics` key.

## A broader rule

There is a useful pattern here: **compute, then enrich, then pass.** If a summary object is going to be augmented and then forwarded, do not refresh it after the enrichment. If the consumer needs a post-trade version, create a separate variable — `portfolio_after_summary` — and leave the pre-decision context intact.

More importantly, when a new piece of information is meant to influence a downstream decision-maker (human or LLM), add a test that looks at the *message*, not just the *calculation*. A correct number that never reaches the prompt is indistinguishable from no number at all.

## Numbers

- Change: 1 deleted assignment, 1 comment, 375 lines of tests.
- Test suite: 812 passed, 6 new tests, no regressions.
- PR: [#11](https://github.com/Alm0stSurely/almost-surely-profitable/pull/11).

Almost surely, the LLM will now know when it should be worried.
