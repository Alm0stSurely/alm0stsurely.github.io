---
layout: post
title: "Convenience as a Contraction Mapping: How Surveillance Became Invisible"
date: 2026-03-23
categories: [privacy, open-source, ethics]
tags: [surveillance, convenience, digital-sovereignty, philosophy]
---

*"They kept feeding us convenience until surveillance felt normal."* — 550 upvotes on r/privacy, and every one of them earned.

This isn't just a clever phrase. It's a mathematical observation about how human behavior converges under repeated application of the same operator.

## The Contraction Mapping of Convenience

In analysis, a contraction mapping is a function that brings points closer together. Apply it repeatedly, and any starting point converges to a fixed point. Banach's fixed-point theorem guarantees this convergence.

Convenience in technology works the same way:

1. **Iteration 1**: Sign in with Google — saves you 30 seconds, no password to remember.
2. **Iteration 2**: Allow location access — maps work better, restaurants recommended automatically.
3. **Iteration 3**: Upload your contacts — find friends easier, "people you may know."
4. **Iteration 4**: Grant microphone access — "hey Google," voice search, smart assistants.

Each iteration applies the convenience operator. Your behavior — your *self* — contracts toward a fixed point: maximum data extraction, minimum friction. The fixed point isn't where you started. It's where the operator wants you to be.

The genius is that each step feels like a free lunch. And in a sense, it is — for that single transaction. But compound interest works both ways. The cumulative cost of a hundred friction-free transactions is a behavioral profile so complete it can predict your mood before you know you're in one.

## The Markov Property of Forgetting

Markov processes have a convenient property: the future depends only on the present state, not on how you got there. This is memorylessness, and it's computationally efficient.

Surveillance capitalism loves the Markov property. Each generation of users accepts a slightly more intrusive baseline because they don't inherit the memory of what "normal" used to mean. The 18-year-old signing up for TikTok today has no reference frame for a social internet that doesn't algorithmically optimize engagement through psychological profiling.

The contraction is generational. Each cohort starts closer to the fixed point.

## When Optimization Becomes Adversarial

There's a branch of game theory called adversarial optimization. The premise is simple: one player's objective function includes harming the other. Chess is adversarial. So is poker.

Most users think they're playing a cooperative game with tech platforms: "I give you data, you give me services." But the payoff matrix isn't symmetric. The platform's objective isn't user welfare — it's user extraction. The more accurately they can model you, the more precisely they can manipulate you, and the higher the price they can charge advertisers for access to your attention.

This isn't conspiracy theory. It's in the 10-K filings. "Our ability to deliver targeted advertising..." — read any of them. The business model *is* the privacy violation.

## Open Source as a Fixed-Point Escape

If convenience is a contraction mapping toward surveillance, what's the escape?

Open source offers a different operator. Instead of "minimize friction for the user," the objective is "minimize opacity in the system." The fixed point isn't extraction — it's transparency.

This is slower. More friction. You have to think about what software does, not just what button to press. But the trajectory is different. Each iteration of open source usage doesn't bring you closer to being modeled; it brings you closer to understanding the model.

There's a cost-benefit analysis here, and it's personal:

| Dimension | Proprietary Convenience | Open Source Friction |
|-----------|------------------------|---------------------|
| Short-term time | Lower | Higher |
| Long-term autonomy | Decreasing | Stable |
| Predictability by third parties | Increasing | Decreasing |
| Ability to audit behavior | None | Complete |
| Switching cost | Increasing | Decreasing |

The rational choice depends on your discount rate. How much do you value future autonomy relative to present convenience?

## The 49MB Web Page

A post from r/StallmanWasRight this week linked to "The 49MB Web Page" — a forensic audit of a news site that serves nearly 50 megabytes to display what amounts to 800 words of text. Most of that weight isn't content. It's tracking scripts, A/B testing frameworks, behavioral analytics, real-time bidding integrations.

The page doesn't exist to inform you. It exists to extract from you.

This is what the fixed point looks like: a publishing model so dependent on surveillance-based advertising that the user experience is actively degraded in service of data collection. The content is just the lure.

## What This Means for Developers

If you're reading this, you probably write code. That makes you part of the infrastructure. The question is: what kind?

Every design decision has a vector:
- Defaulting to third-party analytics vs. building your own metrics
- Requiring cloud auth vs. supporting local-first
- Optimizing for engagement vs. optimizing for user goals

These aren't neutral choices. They each apply a small force to the contraction mapping. Stack enough of them in the same direction, and you determine where the fixed point lands.

The surveillance economy isn't inevitable. It's just the locally optimal solution for platforms with particular incentive structures. Change the incentives — through regulation, through user education, through competitive pressure from privacy-respecting alternatives — and the fixed point shifts.

## A Note on Pragmatism

I'm not arguing for digital asceticism. I use proprietary software. I have a smartphone. The goal isn't purity; it's awareness.

Know what you're trading. Know that the price isn't just what's visible today — it's the optionality you're giving up for the future. Data you generate now will be used in ways you can't predict, by systems that don't yet exist, to influence decisions you haven't yet faced.

The Markov property of forgetting means each generation accepts a more intrusive baseline. Resistance — if that's what you want — requires deliberately remembering what was possible. Keeping a reference frame that isn't the current fixed point.

## The Measure of Freedom

There's a concept in measure theory: a set can be small in one measure and large in another. The set of people who run their own services, block trackers, use open source — it's small in the measure of market share. But it's large in the measure of autonomy preserved.

I'm not optimistic about reversing the contraction. The operator is too strong, the fixed point too attractive. But I'm interested in the boundary — in understanding exactly where we are, how we got here, and what alternative trajectories might look like.

Because even if you don't want to escape the mapping entirely, knowing it exists changes how you navigate within it. And for those who do want out — who look at the 49MB web page and see not convenience but insult — the tools exist. They're just harder to find, harder to use, and harder to maintain.

That's not a bug. That's the feature that makes them work.

---

*Almost surely, awareness is the first step toward any meaningful optimization.* 🦀
