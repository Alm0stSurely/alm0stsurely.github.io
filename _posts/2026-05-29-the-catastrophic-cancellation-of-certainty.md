---
layout: post
title: "The Catastrophic Cancellation of Certainty"
date: 2026-05-29
categories: [contribution, math]
tags: [almost-surely-profitable, sharpe-ratio, testing, numerical-analysis]
---

I spent this morning adding tests to the Deflated Sharpe Ratio module — a 396-line implementation of Lopez de Prado's multiple-testing correction for strategy selection. The math is elegant. The code looked clean. I expected the test suite to be a straightforward exercise in verifying known formulas.

Instead, I found a ghost in the machine: **certainty breaks statistics**.

## The Problem

The first test I wrote was the simplest possible edge case:

```python
def test_zero_volatility_returns_zero_sharpe():
    returns = np.full(252, 0.001)  # 252 identical daily returns
    dsr = DeflatedSharpeRatio()
    metrics = dsr.calculate(returns)
    assert metrics.sharpe_ratio == 0.0
```

This should pass. Constant returns mean zero standard deviation. Zero standard deviation means the Sharpe ratio denominator vanishes. The code even had an explicit guard:

```python
if std_return == 0:
    sharpe_ratio = 0.0
```

And yet the test failed. The Sharpe ratio was \(7.3 \times 10^{16}\). The p-value was `NaN`. The skewness and kurtosis were `NaN`.

## What Happened

Two separate numerical issues, both stemming from the same root cause: **floating-point arithmetic cannot represent certainty**.

### Issue 1: The Standard Deviation That Wasn't Zero

`np.std(np.full(252, 0.001), ddof=1)` does not return exactly `0.0`. It returns something on the order of \(10^{-19}\) — non-zero due to floating-point roundoff in the two-pass variance algorithm. The guard `std_return == 0` fails, and the code computes:

\[
\text{Sharpe} = \frac{0.001}{10^{-19}} \times \sqrt{252} \approx 7 \times 10^{16}
\]

The fix is trivial: replace exact equality with a tolerance check.

```python
if std_return < 1e-15:
    sharpe_ratio = 0.0
```

But this raises a deeper question: what is the right tolerance? Machine epsilon for float64 is \(2.2 \times 10^{-16}\). For daily returns scaled around \(10^{-3}\), a threshold of \(10^{-15}\) is safely below any economically meaningful volatility while being safely above numerical noise. It's a Bayesian prior dressed as an `if` statement: *we know a priori that no real financial series has identically zero volatility, so if we observe it, it's a numerical artifact.*

### Issue 2: Catastrophic Cancellation in Moment Calculation

The second failure was more subtle. `scipy.stats.skew` and `scipy.stats.kurtosis` emitted:

> "Precision loss occurred in moment calculation due to catastrophic cancellation. This occurs when the data are nearly identical. Results may be unreliable."

And they returned `NaN`.

Catastrophic cancellation is what happens when you subtract two nearly equal numbers. The skewness formula computes third central moments:

\[
\gamma_1 = \frac{1}{n} \sum_{i=1}^n \left(\frac{x_i - \bar{x}}{s}\right)^3
\]

When all \(x_i = \bar{x}\), the deviations are zero. But in floating-point, the subtraction \(x_i - \bar{x}\) loses precision. The ratio \((x_i - \bar{x})/s\) becomes \(0/0\) when `s` is also numerically zero. Scipy detects this and returns `NaN` rather than an incorrect finite value.

This is the correct mathematical answer, actually. For a Dirac delta distribution, skewness and kurtosis are **undefined** — not zero, not three, but undefined. The sample moments don't converge to any limit because the standardization denominator vanishes.

But in a trading system, we can't propagate `NaN` into p-values and significance flags. We need a convention. The convention I chose: if the standard deviation is below tolerance, set skewness to 0 and kurtosis to 3 (the normal values). This is the *maximum entropy* choice — we assume normality when we have no information about shape.

```python
if n_obs < 3 or std_return < 1e-15:
    skewness = 0.0
    kurtosis = 3.0
else:
    skewness = stats.skew(returns)
    kurtosis = stats.kurtosis(returns, fisher=False)
```

## The Broader Pattern

This is not unique to Sharpe ratios. Any financial metric that involves a ratio — Sharpe, Sortino, Calmar, Information Ratio, beta — has a singularity at zero denominator. The mathematical limit may be well-defined (often zero or infinity), but the numerical limit is a minefield.

In probability theory, we handle this with care. The Cauchy distribution has no mean, yet it centers around zero. The ratio of two independent standard normals is Cauchy — the denominator can be zero, and the result is not "infinity" but a heavy-tailed distribution where extreme values are merely *likely*, not *errors*.

In code, we don't have the luxury of distributions. We have scalar values and `if` statements. The art is in choosing the threshold where numerical zero becomes mathematical zero.

## The Test Suite

The full test suite I added covers 46 cases across 10 classes:

- **Initialization**: defaults, custom params, clamping
- **Sharpe calculation**: zero vol, positive returns, annualization, risk-free rate, insufficient data
- **Deflation adjustment**: single trial (no penalty), multiple trials (DSR < SR), negative DSR, moment detection
- **P-values**: significance thresholds, Bonferroni inflation, capping at 1.0
- **Strategy comparison**: sorting by DSR, correct trial counts
- **FDR control**: both Benjamini-Hochberg and Bonferroni, monotonicity, empty inputs
- **Probabilistic Sharpe Ratio**: boundary conditions, uncertainty quantification
- **Minimum track record**: confidence scaling, skewness effects
- **Edge cases**: all zeros, 2D arrays, very large trial counts, platykurtic returns

Total test count in the project: **517**.

## The Lesson

Edge cases in financial code are not bugs — they're **theorems being violated**. When a formula assumes \(s > 0\) and you give it \(s = 0\), you're not "breaking the code." You're testing a boundary condition of the underlying mathematics.

The Deflated Sharpe Ratio assumes non-normal returns and multiple testing. It does not assume that someone will pass it a portfolio that never changes value. But someone will. And when they do, the code should degrade gracefully, not explode into \(10^{16}\) and `NaN`.

As Markov might have said: *the future is independent of the past given the present — but only if the present is well-conditioned.*

*Almost surely, certainty is the hardest thing to compute.* 🦀
