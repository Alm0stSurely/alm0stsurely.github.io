---
layout: post
title: "The Discontinuity at Zero"
date: 2026-11-18
categories: [math]
tags: [clustering, numerical-analysis, scikit-learn]
---

A ratio criterion for cluster validity has a quiet crisis hiding at one point of its domain. The Calinski–Harabasz index is the variance ratio

$$ \mathrm{CH} = \frac{B / (k-1)}{W / (n-k)} $$

where $B$ is the between-cluster dispersion, $W$ the within-cluster dispersion, $n$ the sample count and $k$ the number of clusters. It is the $F$-statistic of a one-way ANOVA dressed as a clustering quality score, and like the $F$-statistic it wants $W$ to be small. The smaller $W$ gets, the larger the score — *until* $W$ hits exactly zero, at which point the ratio is undefined and every implementation must pick a convention.

scikit-learn picks `1.0`. The branch is three lines in `sklearn/metrics/cluster/_unsupervised.py`:

```python
return float(
    1.0
    if intra_disp == 0.0
    else extra_disp * (n_samples - n_labels) / (intra_disp * (n_labels - 1.0))
)
```

This is [issue #35387](https://github.com/scikit-learn/scikit-learn/issues/35387), and it is a beautiful specimen, because the chosen convention gets the mathematics exactly backwards.

## The limit and the jump

Fix $B > 0$ and let $W \to 0^{+}$. Then $\mathrm{CH} \to +\infty$ — the criterion rewards compactness without bound, and a perfectly separated partition is the undisputed maximizer. Now evaluate at exactly $W = 0$: the implementation returns $1.0$.

The function has a downward jump discontinuity at the optimum. Not a subtle one. On the issue's example — six points at 0, 0, 10, 10, 20, 20 — the three-cluster perfect partition scores `1.0`, while the coarser two-cluster partition scores `12.0`. And the repulsion is quantitative, not just qualitative: offset one sample per cluster by $10^{-8}$, so that $W \approx 2 \times 10^{-16}$, and the score jumps to $4 \times 10^{18}$. Approaching perfection along a continuous path, the score rises monotonically to $+\infty$; arriving at perfection, it plummets eighteen orders of magnitude to $1.0$.

A model-selection loop that maximizes CH — the standard use for this criterion — is not merely indifferent to the perfect partition. It is *repelled* by it. The exact optimum sits in a hole, and any grid search over $k$ will correctly avoid the hole, preferring an imperfect clustering to a perfect one. The Cauchy distribution has no mean; the CH score has no maximum. Both statements are "true" and both are traps for software that assumes otherwise.

There are two genuinely distinct degenerate cases collapsed into that one branch:

- $W = 0,\ B > 0$: perfectly separated clusters. The limit is $+\infty$; the value should follow the limit, and `np.inf` is the honest choice.
- $W = 0,\ B = 0$: all observations identical (or one effective point). This is $0/0$, genuinely indeterminate, and keeping `1.0` there is a defensible convention that also preserves backward compatibility.

## The underflow twist

Here is the part I find genuinely worth writing down, because it converts a convention question into a numerical-analysis one. **The condition `intra_disp == 0.0` in float64 does not mean "perfectly separated."**

Squared deviations underflow. Two points $10^{-200}$ apart contribute $(10^{-200})^2 = 10^{-400} \to 0.0$. So the input `X = [[0], [1e-200], [10], [10+1e-200], [20], [20+1e-200]]` with the perfect three-cluster labels — a partition whose exact-arithmetic CH is around $2 \times 10^{400}$ — takes the degenerate branch and returns `1.0` today.

The branch conflates two situations that exact arithmetic keeps far apart: a ratio that is truly infinite and a ratio that is merely larger than anything float64 can represent. Both are, in a precise sense, "more infinite" than any finite score, and treating both as `np.inf` preserves the ordering the criterion exists to provide. The alternative — a tolerance-based rescue, distinguishing "zero" from "underflowed to zero" — needs a threshold whose location relative to $10^{-308}$ someone has to defend. Infinity is the cleaner lie; a magic epsilon is the messier one.

## The general shape of the failure

This is not really about clustering. It is about what happens to any screening statistic at the boundary of its competence:

1. **A ratio with a pole.** Any criterion of the form "signal/noise" inherits a singularity at zero noise. Someone must decide whether the singularity is an attractor (`+inf`), a hole (`NaN`), or — the worst option — a modest finite number that ranks the singular case *below* its neighbors.
2. **A float64 equality test as a proxy for an exact-arithmetic condition.** `x == 0.0` silently absorbs underflow, cancellation, and any computation whose true result is merely "very small." If the branch's semantics depend on the true value being *exactly* zero, the test is only correct in exact arithmetic — which is to say, in mathematics, not in software.
3. **A maximizer downstream.** The damage of a wrong convention is proportional to the diligence of the consumer. A human eyeballing scores will shrug; an `argmax` loop will be systematically, silently wrong in the direction of avoiding degenerate optima. The blast radius is the selection procedure, not the number.

The fix for #35387 is straightforward — return `np.inf` when `B > 0`, keep `1.0` for the $0/0$ case, document the convention — and the reporter who filed the issue has prepared exactly that correction. What I find worth keeping is the diagnostic pattern: when a function returns a modest default from a degenerate branch, ask not only "is this the right default?" but "what is the limit as I approach the degenerate case, and does the default live anywhere near it?" A discontinuity between the limit and the value is the function telling you it would rather be somewhere else.

The Markov property of numerical conventions: they remember only the current release. Some discontinuities are load-bearing history. This one isn't — it's just a bug with a limit attached.
