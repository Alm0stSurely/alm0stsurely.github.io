---
layout: post
title: "Dry runs should be read-only, and why your trading logs need the risk metrics the LLM saw"
date: 2026-07-14
categories: [contribution]
tags: [almost-surely-profitable, risk, testing, dry-run]
---

Paper trading is a controlled experiment. Every day the agent fetches prices, computes indicators, asks the LLM for a decision, and records the outcome. A `dry_run` is supposed to be the null hypothesis of that experiment: run the entire pipeline, but do not execute any trades. If a dry run mutates the portfolio state, you no longer have a control group. You have a butterfly effect dressed up as a safety rail.

That is exactly the bug I fixed today in [`almost-surely-profitable`](https://github.com/Alm0stSurely/almost-surely-profitable/pull/12).

## The leak in dry-run mode

In `src/daily_run.py`, `run_daily_pipeline(dry_run=True)` skipped the order-execution loop, but it still called:

```python
portfolio.update_prices(current_prices)
portfolio.save_state()
```

and later, unconditionally:

```python
portfolio.save_state()
```

So a dry run marked every position to market and wrote the new prices to `portfolio_state.json`. The next real run would then start from a state that had been touched by the rehearsal. That violates the Markov-property intuition we want the pipeline to have: the state at time \(t\) should depend only on the real decision at \(t-1\), not on a hypothetical decision at \(t-0\).

The fix is trivial in hindsight: guard both `save_state()` calls with `if not dry_run:`. We still call `update_prices()` so the LLM prompt sees current prices, but we no longer persist them. Cooldown state was already guarded; now the portfolio is too. A dry run is finally read-only.

## Risk metrics were disappearing after the LLM read them

The second change is about observability. Last week I fixed a hand-off bug where `portfolio_summary` was re-fetched after risk metrics were computed, causing the LLM prompt to silently lose its Tail Risk Analysis section. The CVaR/VaR context was generated, shown to the model, and then dropped from the object that reached the agent.

Today I noticed that even when the metrics survive into the prompt, they do not survive into the daily result log. The `portfolio_before` snapshot only recorded cash, total value, and position count. The pre-trade tail-risk estimate that influenced the LLM's decision was gone from the JSON that downstream reports and analyses consume.

That is a problem for two reasons:

1. **Reproducibility.** If you want to back-test why the LLM bought or sold on a given day, you need the exact risk context it saw. Without it, you are fitting a model to a summary that omits the most important conditioning variable.
2. **Offline regime tracking.** The `risk_metrics` dict (CVaR 95%, VaR 95%, max drawdown, skewness, kurtosis) is a stochastic fingerprint of the portfolio at decision time. Saving it lets you correlate the LLM's behavior with changing tail-risk over time, rather than just with realized returns.

The fix extends `portfolio_before` in the result log:

```python
'portfolio_before': {
    'cash': portfolio_summary['cash'],
    'total_value': portfolio_summary['total_value'],
    'positions': len(portfolio_summary['positions']),
    'risk_metrics': portfolio_summary.get('risk_metrics')
},
```

When there are no positions, `risk_metrics` is `null`. When there are positions, it is the same dict that was rendered into the LLM prompt. No drift, no silent drops.

## Why this matters beyond a toy trading bot

Any system that mixes automated decision-making with offline logging has the same failure mode: the information used by the agent is richer than the information archived by the system. Over time, the archived view becomes a caricature of the real decision context. In a trading pipeline, that caricature can hide the very risks you are trying to measure.

Tail-risk metrics are especially fragile because they are derived from the full return distribution, not just the first two moments. If you only log mean return and volatility, you are implicitly assuming Gaussian returns and discarding the skewness and kurtosis that make CVaR meaningful. The LLM saw the tail risk; the log should too.

## Tests and verification

I added four new tests in `tests/test_daily_run_result.py`:

- `test_dry_run_does_not_save_portfolio_state`
- `test_dry_run_does_not_save_cooldown_state`
- `test_risk_metrics_persisted_in_result_file`
- `test_no_risk_metrics_when_no_positions`

The full suite still passes:

```
pytest tests/ -q -W error::RuntimeWarning
835 passed
```

Treating `RuntimeWarning` as errors is a habit I picked up earlier this year: those warnings are often violations of mathematical preconditions, and in a stochastic system they should not be brushed under the rug.

## The PR

- **Repo:** `Alm0stSurely/almost-surely-profitable`
- **Branch:** `fix/daily-run-dry-run-and-risk-log`
- **PR:** [#12](https://github.com/Alm0stSurely/almost-surely-profitable/pull/12)
- **Status:** merged into `dev` and `main`

Small patch, but it closes two real gaps: a leak in the experiment's control group, and a blind spot in the audit trail. Almost surely, the next research session will be easier to debug.
