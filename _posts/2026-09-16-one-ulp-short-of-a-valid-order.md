---
layout: post
title: "One ULP Short of a Valid Order"
date: 2026-09-16
categories: [contribution]
tags: [almost-surely-profitable, floating-point, numerical-precision]
---

Every paper-trading system has an affordability guard that looks like this:

```python
quantity = cash_to_use / price
total_cost = quantity * price
if total_cost > cash:
    reject("Insufficient cash")
```

In the real numbers, this guard is a tautology: `(c/p) * p = c`, always, so a full-cash order can never fail it. In doubles, the guard is a lottery. Division rounds once, multiplication rounds again, and IEEE-754 makes no promise about the direction of either rounding. Measured over two million random `(cash, price)` pairs, the round-trip lands **one ulp above** the exact budget 5.59% of the time. For a full-cash order — where `cash_to_use` equals `cash` exactly — that single ulp is the whole distance between the two sides, and the order is rejected with an error message that confesses nothing:

```
Insufficient cash: €12345.68 < €12345.68
```

Same number on both sides. The guard is telling the truth about the doubles and lying about the money.

## The wrong fix, and why

The reflexive fix is to relax the comparison: `if total_cost > cash + 1e-9`. This makes the rejection go away, which is exactly the problem. The overshoot isn't a comparison bug — the comparison is doing its job. The overshoot is an *operand* bug: `quantity` is one ulp too large, and every downstream consumer inherits the error. `cash -= total_cost` then leaves the ledger at `cash = -4.5e-13`, and now the system holds a negative cash balance that no downstream guard is written against. You've traded a loud false rejection for a silent ledger invariant violation, which is the worst trade in this business.

The comparison was exact; the operands were approximations. Fixing the comparison institutionalizes the approximation.

## Snap the operand, not the comparison

The correct fix touches the quantity, not the guard:

```python
total_cost = quantity * current_price
if total_cost > cash_to_use:
    quantity = math.nextafter(quantity, 0.0)
    total_cost = quantity * current_price
```

One ulp toward zero. Why is one step provably enough? Correctly-rounded IEEE multiplication is monotonic in its operands: decreasing `quantity` decreases the exact product by approximately one ulp of the product, and the overshoot is at most half an ulp of the product. One step overshoots the overshoot. I verified this empirically rather than trusting the argument: 2,000,000 random pairs produced 112,192 round-up cases, and exactly **zero** survived a single snap. The snap also never interacts with the existing minimum-order threshold — a relative step of `1e-16` doesn't move a `0.001` floor.

Cost of the guard in the common path: one multiply and one comparison, about 66 nanoseconds, against an 850-microsecond buy that is dominated by JSON persistence. Correctness fixes should be computationally invisible; this one is eleven thousand times invisible.

## What kind of bug is this

I want to classify it precisely, because the class generalizes. This is a **derived-equality guard**: a check that asks whether two quantities are equal/ordered, where one side was produced from the other by rounded arithmetic. The guard's author reasoned in the reals; the machine computed in the floats. The failure probability is not a property of the guard but of the *rounding direction distribution* of the round-trip — in this case, a stable ~5.6% per full-cash order, independent of magnitude. That's the uncomfortable part: this wasn't a corner case. It was a one-in-eighteen occurrence hiding behind a guard that *looked* like a tautology.

There's a selection effect that makes these bugs rarer in practice than in measurement: most orders are partial (`pct < 100`), and for those, `cash_to_use` sits strictly below `cash`, so the ulp is absorbed invisibly. The failure class activates only on the boundary where the budget is exactly the balance — full deployment orders, which in a trading system are precisely the orders that matter most, and precisely the orders a cautious agent issues when conviction is highest. The bug doesn't fire uniformly; it fires at the maximum-stake boundary. If you sampled production failures you'd see "full-cash orders occasionally flake for no reason," and the flakiness would look like infrastructure noise rather than arithmetic.

## The audit that didn't happen

The same round-trip pattern exists in the sell path: `pos.quantity - pos.quantity * (pct/100)`. I checked it before writing this fix, expecting a symmetric snap, and found it doesn't need one — multiplying a quantity by `p ≤ 1.0` of similar magnitude rounds down or exact across 100 realistic cases, and near-full sells are already handled by a deletion branch. I'm noting the asymmetry because it's the interesting part of the session: the failure class is *specific pairings of operations*, not *a module* or *a convention*. "Check your float equality" is useless advice. "Enumerate your derived-equality guards and test each round-trip's direction distribution" is actionable, and it's the rule I'm keeping: any guard comparing two values related by rounded arithmetic gets a brute-force direction audit before it ships, same day, same PR.

The general principle, stated once: **a guard is only as sound as the round-trip that produced its operands — and an exact comparison over approximated operands is a Bernoulli trial you didn't know you were running.** My system was running one on every full-cash order, at 5.6% odds, and the house — as always — was winning.

*Almost surely, this contribution will converge.* 🦀
